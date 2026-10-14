# Android architecture

[← README](../README.md) · [Architecture](ARCHITECTURE.md) · [Research pipeline](RESEARCH_PIPELINE.md) · [Backend](BACKEND_ARCHITECTURE.md) · [Security](SECURITY.md) · [Testing](TESTING.md)

A single-activity app (minSdk 26, target and compile SDK 37) drawn entirely in Compose.

| Concern | Technology |
|---|---|
| UI | Jetpack Compose, Material 3, a single `AppCompatActivity` |
| Dependency injection | Hilt |
| Concurrency | Kotlin coroutines and Flow |
| Database | Room (schema exported and checked in; 16 tables) |
| Settings | DataStore Preferences |
| HTTP | OkHttp, kotlinx.serialization |
| Background work | An app-owned foreground service (manual runs), JobScheduler (daily runs) |
| Credentials | Android Keystore, AES-256-GCM |
| Locale | AndroidX per-app locales |
| Build | Gradle, Android Gradle Plugin, Kotlin, KSP, ktlint, R8 for release |

No Google Play Services dependency: no Firebase, Play Integrity or Play Core.

---

## Presentation layer (Compose)

- **Two destinations.**
  - **Daily** reads results: the topic, the latest report card, one `Start` button, the highlights, then history and secondary rows. It holds no configuration.
  - **Topics** configures future runs: the brief, `Daily update`, the output preferences, the report style, `+ New topic`, `AI service` and `Language`.
- **An interaction hierarchy.** Use the first that stays clear: a direct control; inline expand/collapse; a chip, slider, segmented control or toggle; a bottom sheet for a short choice; a dedicated screen only when the content needs the space. Primary actions take one or two taps, and reversible actions have no confirmation dialog.
- **Navigation.** A small, saveable overlay back stack holds the report (tabs: report, sources, events), Changes, History, AI service and the full topic editor.
- **State.** Controllers observe Room through `Flow`. Settings changes are applied optimistically and written in order, so they show at once.
- **Visual system.** Meaning comes from five tonal colour families (fresh, updated, insight, caution, problem), always next to words that say the same thing. A test pins WCAG AA contrast for every colour pair drawn, in light and dark. Touch targets are at least 48 dp, and layouts are checked at 200 % font scale.
- **Motion.** A few fixed duration tokens (150–240 ms), no springs. Motion explains state changes, and everything works with animations disabled.

---

## Repository boundaries and persistence

The UI never talks to Room or the network directly. It talks to repositories in `:core:data` (topics, reports, settings, engine setup, runs), which expose `Flow`s and suspend functions. The research core reaches storage and the network only through its ports.

**Room (16 tables).** Topics and their revisions, provider configuration, runs and run logs, usage records, evidence sources, events and observations, observation–source links, and reports with their sections, items, trends and metrics. A separate table keeps each run's validated result, compressed, with key-shaped text redacted before compression.

- Instants are stored as UTC epoch milliseconds and dates as epoch days; foreign keys are enforced.
- **The active-run guard** ("insert a run only if none is pending or running") is a single transaction. A test with two database connections shows exactly one of 24 concurrent inserts winning.
- A scheduled slot's identity is a unique column, so a re-delivered job finds the existing run instead of creating a second one.
- Reports are immutable once written.
- The schema is exported. Later changes need an explicit migration with a migration test; destructive fallback is not allowed.

**DataStore** holds the active topic, the selected provider and agent, matching settings, the per-topic report style and a few UI flags.

---

## Secure credential storage

- API keys are encrypted with AES-256-GCM under one non-exportable Android Keystore key, StrongBox-backed when the device has StrongBox. Associated data binds each record to its provider.
- Room stores only "a key is saved" and the last four characters, for display.
- The credential directory is excluded from cloud backup and device transfer. After a restore, start-up reconciliation clears the "saved" flag of any key that can no longer be decrypted, so the user is asked again.
- The key-entry screen blocks screenshots (`FLAG_SECURE`).

More in [Security](SECURITY.md).

---

## Networking

- **One shared OkHttp client** with `retryOnConnectionFailure(false)`: OkHttp would otherwise silently re-send a POST that failed on a reused pooled connection, which for a research call is a second paid request. Each call sets its own timeout from the provider's research budget, and cancelling the coroutine cancels the call.
- **HTTPS only.** Provider base URLs must be `https://`, and cleartext is disabled app-wide.
- **Seven provider adapters** (DeepSeek, Qwen, Zhipu GLM, Kimi, OpenAI, Anthropic Claude, Google Gemini). Each has its own wire format and web-search tool, reads the vendor's stop reason to detect a cut reply, and records a usage row. All are pinned by wire-format fixtures over MockWebServer.
- Release builds never log request or response bodies.

---

## Background execution (JobScheduler and a foreground service)

### How the design was chosen

Before the run hosting was written, a throwaway spike app hosted the same workload in three candidates: WorkManager expedited work, a user-initiated data-transfer job, and an app-owned `dataSync` foreground service. The workload was one HTTP call to a local server that stays silent for 10 minutes and then answers, with a request ledger as evidence of duplicates. They ran on a Galaxy S25 Ultra (Android 16) with the screen off, the app swiped away, the notification permission denied and long background periods.

Findings that became requirements:

- **Hosts are re-dispatched after process death.** WorkManager re-ran a killed worker by itself about 70 s later, and a user-initiated job came back when the app returned to the foreground. A host's retry policy cannot prevent a second paid call.
- **OkHttp silently retried a POST** on a dropped pooled connection.
- **Promotion to a foreground service from the background was refused**, so scheduled runs can't rely on it.

### Manual runs

`Start` inserts the run through the active-run guard, then starts a foreground service (type `dataSync`) with only the run id. The service mirrors the run's stage in its notification and stops itself when no run is left. If the system stops it, the run is recorded as failed with an "interrupted, please retry" message.

### Daily runs

One waiting JobScheduler job per scheduled topic, with a minimum latency until the next local time, a network requirement, and persistence across reboots. The schedule is reconciled at every process start, on every schedule change and after a time-zone or clock change.

**The job owns its run.** The job service starts the run and reports completion only when the run has ended, so the platform keeps the process protected for the whole call. An earlier design ran the work in-process through WorkManager; on a real device that finished the platform job early, and the cached process was frozen within seconds and later killed.

- When the system stops the job, the run is cancelled and recorded as failed. Nothing is rescheduled or retried.
- A firing gets a fixed time budget, inside the platform's job window.
- The battery-optimization exemption is offered, never forced, when the daily update is turned on. Daily runs are best-effort, and the time is shown as "around HH:MM".

### No duplicate paid calls, by construction

```kotlin
// Illustrative sketch, simplified; not the production source.
suspend fun execute(runId: RunId) {
    // Atomic pending -> running, before any network access.
    if (!store.claim(runId)) return       // already running, finished, or failed by the reconciler
    try {
        val reply = agent.research(request) // the primary research request; the HTTP client never retries
        // ... validate, recover, at most one repair request, record events, assemble the report
    } catch (e: CancellationException) {
        withContext(NonCancellable) { store.fail(runId, interrupted = true) }
        throw e
    }
}
```

- **A reconciler** at every process start fails any pending or running run with no live host in this process. A run is reserved the moment it is created, so a reconcile that races a host still starting can't fail it.
- **Hosts never retry.** Nothing re-runs a started run.
- **What a run may send.** One primary research request, at most one repair request, and, for ambiguous event matches only, short bounded matching calls ([Research pipeline](RESEARCH_PIPELINE.md#5-evidence-and-event-processing)).

**Device evidence.** On the Galaxy S25 Ultra, on simulated battery with forced Doze and the exemption granted, one scheduled slot produced exactly one run and one agent execution: two vendor requests (the primary request and its one permitted repair), no scheduling duplicates and no automatic retries.

### Notifications

A silent channel for run progress, one result notification per topic, and a "didn't finish · tap to retry" message for a failed scheduled run. The channel is created before every post: an emulator showed that a job's fresh process could post before the channel existed, and Android dropped the notification. The notification permission is requested in context, the first time the user starts a run, and runs work without it.

---

## Report rendering

- **On screen:** native Compose over structured rows, with one presenter holding every text rule. The screen, the Daily highlights and the export share it. Tabs: the report, its sources (primary first, then background) and its events (each opens a timeline).
- **Export:** a self-contained HTML document with inline CSS, a Content-Security-Policy, every value escaped and links only for `http(s)` addresses. Four presentation-only styles. Shared through a `FileProvider`, or saved through the system file picker.
- **Links** are re-checked for scheme when opened, in a Custom Tab where available.

---

## Per-app localization

- **Resources.** English is the default resource language; Simplified Chinese is a translation, with matching placeholders and plurals.
- **Mechanism.** `AppCompatDelegate.setApplicationLocales`. On Android 13+ the platform stores and applies the choice, and the system settings also offer it. On Android 8–12, AppCompat stores it and recreates the activity.
- **Text outside an activity.** A foreground service, or a JobScheduler job in a process with no activity, resolves text through a locale-aware context, so scheduled-run notifications are localized on every supported version.
- **Domain values stay language-neutral.** Enums keep stable codes and are mapped to resources at the UI boundary. Stored research-layer messages never change; recognised ones are re-rendered in the UI language, and unknown ones are shown as written.
- **UI language ≠ report language.** Nothing in the UI language reaches a prompt, a stored value or a report.
- **Tests** fail on a missing translation, a placeholder mismatch, a hard-coded string in Compose code, or Chinese chrome in the English UI. A test also checks that an exported report is byte-identical in both UI languages.
- **Validated on device:** the English and Chinese UI and switching between them, on a Galaxy S25 Ultra (Android 16).

---

## Account client (implemented, not wired)

A complete client for the backend's API exists in `:core:data`, but **no screen uses it**, and a test pins that, so the app makes no account request today.

- **Installation id:** a random UUID in no-backup storage, never transferred. No hardware identifier is used.
- **Session storage:** the refresh token sealed with the Keystore key under its own associated data; the access token only in memory.
- **Refresh:** single-flight (every caller joins the one refresh in flight, because the backend treats a rotated token presented twice as replay) and bounded (a 401 leads to at most one refresh and one retry).
- **The backend address** is a per-build-type value from local configuration. It is empty by default, and then nothing is sent.
