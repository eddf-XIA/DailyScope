# Testing

[← README](../README.md) · [Architecture](ARCHITECTURE.md) · [Android](ANDROID_ARCHITECTURE.md) · [Research pipeline](RESEARCH_PIPELINE.md) · [Backend](BACKEND_ARCHITECTURE.md) · [Security](SECURITY.md)

Most of DailyScope's behavior is decided in pure Kotlin, so most of it is tested on a plain JVM: fast, deterministic, and with no device. UI and persistence tests also run on the JVM, through Robolectric on the real data stack. A small set of checks needs real Android: the Keystore, Android's regex engine, and background execution on hardware.

---

## Recorded results

From a recorded full run on 2026-10-06: JDK 21, no build cache, no local machine configuration, no credentials and no paid calls.

| Kind | Suite | Result | What it covers |
|---|---|---|---|
| **Unit** (pure JVM) | Research core | 1,023 passed, 1 opt-in skip | Golden and specification fixtures, the runner, prompts, validation, recovery, identity, matching, changes, report HTML |
| **Integration** (Robolectric) | Android data layer | 388 passed, 2 live-provider tests skipped | Room, DataStore, the credential store, the seven provider adapters over MockWebServer, the chat helper, run coordination, the account client |
| **UI** (Robolectric) | Android app | 212 passed, 1 opt-in skip | Compose screens on the real stack, run hosting and scheduling, notifications, localization, contrast, performance |
| **Screenshot comparison** (Roborazzi) | Report scenes, from 4 of the 212 UI tests | 48 of 48 images match their baselines | 12 report scenes × light/dark × default/200 % font |
| **Database integration and concurrency** | Backend, on real PostgreSQL 17 | 66 passed | API, activation, sessions, devices, entitlements, admin CLI, migrations, secret logging |
| **Instrumented** | Real Android Keystore; research-core suite on ART | 2 tests passed; 965 research-core checks passed on ART | Run on an API 34 emulator |

**1,689 test cases** across the four JVM suites (1,023 + 388 + 212 + 66). The screenshot comparisons and the instrumented checks are not added to that: they come from tests already counted, or re-run them on ART. The same run also built the debug APK, ran Android lint with no errors, and passed ktlint in both builds.

**Since that run**, the only test-code change is five opt-in tests that render this portfolio's screenshots; they are skipped by default and not included above.

**Live tests.** Two data-layer tests call DeepSeek for real. They are skipped unless explicitly enabled, because they are billable.

---

## Fixture-driven parity

The research core is a port of DailyScope's Python predecessor, AIOS Intelligence Monitor, and its behavior is pinned by data rather than by prose. Fixture cases are test data, not extra tests: they drive tests counted in the suites above.

| Fixtures | Cases | Origin |
|---|---|---|
| Golden | **737** in 15 families | Generated from the unmodified Python app by a deterministic harness (frozen clock and time zone; two generations must be byte-identical) |
| Android specification | **94** | Hand-authored for Android-only behavior (output preferences, local JSON recovery) |

The golden families cover normalization, canonical URLs and hashing, event keys, identity fingerprints, event matching, change classification, JSON extraction, result validation, briefs, prompts, engine readiness, secret masking and redaction, provider wire formats, whole runs, and report views.

Rules:

- **When Kotlin disagrees with a golden case, Kotlin is wrong**, unless a recorded design decision says otherwise. Then the case carries an explicit override with its reason, so intentional differences are visible in the data, not hidden in code.
- **The two fixture classes are never mixed.** Golden files come only from the generator; specification files are only hand-authored.
- **Case ids are never renumbered.**

The same fixtures drive several layers: whole-run cases run through the pure runner and again through the Room-backed store; report-view cases are checked on the Compose screen; wire-format cases drive each provider adapter over MockWebServer.

---

## Selected tests and what they protect

| Test | Guarantees |
|---|---|
| Active-run guard | Holds across two database connections: 1 of 24 concurrent inserts wins |
| Run lifecycle | Claim-once, cancellation and interruption over a real provider adapter and MockWebServer: no second request |
| Scheduled job | The JobScheduler job owns its run; a stop cancels and never reschedules; re-deliveries are refused by slot identity |
| Request invariance | 27 output-preference combinations × 7 agents: the request bodies differ only in the prompt text |
| Preference isolation | Output preferences never change event identity, matching or change detection |
| Reply robustness | Per-vendor truncation detection; local recovery before the one repair, for all seven agents |
| Credential leak | A canary key appears nowhere in plain text in the app's data directory |
| Session repository | Single-flight, bounded refresh; tokens never leak into state or errors |
| Account not wired | Nothing in the app calls the account client yet |
| Localization | Complete, consistent translations; no hard-coded UI text; the exported report is identical in both UI languages |
| Contrast | WCAG AA for every colour pair drawn, light and dark |
| Large history | 1,050 events and 57 reports through the real runner: views stay fast, and runs don't slow down with history |
| Backend concurrency | Concurrent activation of one code creates one account; concurrent refreshes with one token yield at most one new session and then revoke it; concurrent grants leave one open entitlement |
| Backend migrations | Migrations apply once on an empty database; constraints and the append-only audit log hold |
| Backend secret logging | Sign-in, refresh, replay and logout log no credential |

**Backend concurrency is tested against a real PostgreSQL**, never an in-memory fake. The tests start embedded PostgreSQL 17 binaries, so no Docker is needed.

**Mutation checks.** For the critical guards, a deliberate break was introduced by hand to confirm that a test fails: for example a per-delivery slot key instead of a per-slot one, a job stop that doesn't cancel, a Chinese literal in Compose code, and a translated error text off by one character. These were one-off manual checks; there is no mutation-testing tool in the build.

---

## Emulators and a physical device

Outside the automated suites:

- **Emulators (API 26, 30, 34 and 37):** the R8 release build installs and runs, the real-Keystore instrumented test passes, and the research-core suite runs on ART.
- **One physical phone, a Galaxy S25 Ultra (Android 16):**
  - the background-execution spike that chose the run hosts;
  - a daily-schedule check on simulated battery with forced Doze: one slot, one run, one agent execution, two vendor requests (the primary request and its one permitted repair), no duplicate request and no automatic retry;
  - walkthroughs of the main flows, including real, user-approved research runs;
  - the English and Simplified Chinese UI and switching between them.
- **Not yet done:** manufacturer-specific power management (OEM ROMs), a fully unplugged natural-idle scheduled run, and a physical TalkBack walk.

## How the screenshots were made

The README's images come from the app's own Compose UI on the real data stack, rendered on the JVM by Robolectric and captured with Roborazzi, from fictional test data. No device, account or personal data is involved.
