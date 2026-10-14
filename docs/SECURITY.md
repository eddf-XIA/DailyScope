# Security

[← README](../README.md) · [Architecture](ARCHITECTURE.md) · [Android](ANDROID_ARCHITECTURE.md) · [Research pipeline](RESEARCH_PIPELINE.md) · [Backend](BACKEND_ARCHITECTURE.md) · [Testing](TESTING.md)

What DailyScope protects, how, and what it does *not* claim. Everything below is implemented unless marked otherwise.

## What needs protecting

| Asset | Where it lives |
|---|---|
| The user's AI provider API keys | On the device, encrypted |
| Paid provider calls | Made from the device with the user's key |
| Research output (reports, sources) | On the device; never uploaded |
| An account session (future use) | The refresh token on the device, encrypted; only its hash on the backend |
| Backend secrets (database password, token signing secret) | Server environment variables only |

---

## Provider API keys (Android)

- **Encryption.** AES-256-GCM under one non-exportable Android Keystore key, StrongBox-backed when the device has StrongBox, otherwise hardware-backed through the TEE. Associated data binds each record to its provider, so a record can't be moved to another provider or opened as another kind of secret.
- **No user authentication on the key, by design.** Scheduled runs must decrypt in the background with no user present. The trade-off: the key protects data at rest and against extraction, not against code running inside the app's own process.
- **Room never stores a key**: only "a key is saved" and the last four characters, for display.
- **A narrow key flow.** The key goes from the input field to the credential store, and is read only immediately before a request. It never goes into an intent, job extras, saved state, navigation arguments, logs, notifications or error text.
- **Not in backups.** The credential directory is excluded from cloud backup and device transfer. After a restore, keys that can't be decrypted are detected and the user is asked again. A *temporarily* unavailable Keystore never clears anything.
- **The key-entry screen** blocks screenshots and the recents thumbnail, and shows a saved key only masked.
- **Over the wire**, keys travel only in headers over HTTPS, never in a URL.

## Redaction and logging

- A redaction step replaces token-shaped strings and every known secret before any text is written to a run log, an error message or a usage record, shown in the UI, or logged.
- Stored result blobs are redacted **before** compression, so a scan of the data directory still means something.
- **A leak test** stores a canary key, exercises the data layer, and then scans every file in the app's data directory, including the databases, for it in plain text.
- Release builds never log request or response bodies.

## Transport and untrusted content

- **HTTPS only**, including on Android 8.0–8.1, where the platform default still allows cleartext. The backend address must also be `https://`, and the build checks it.
- **Model output is untrusted input.** On screen it is plain Compose text, never HTML or Markdown, and there is no WebView. The HTML export escapes every value, inlines its CSS, carries a Content-Security-Policy and links only `http(s)` addresses. Source links are re-checked when opened.

## Bounded spending

A research call costs the user money. A run makes one primary research request and at most one repair request, plus short, bounded matching calls only for ambiguous events; network and HTTP errors on research requests are never retried; an interrupted run ends as failed and only the user can start another. A claim-once transition, a unique scheduled-slot identity, a start-up reconciler and an HTTP client with silent retries disabled stop Android's host re-dispatch from repeating a call.

---

## Account sessions

### On the device (client implemented; not used by the app yet)

- The refresh token is sealed with the Keystore key in its own file, under its own associated data. An account record never opens as an API key, or the other way round.
- The access token is held only in memory.
- Session files are excluded from backup, so a restored or transferred app starts signed out.
- The installation id is a random UUID in no-backup storage. No hardware identifier is read.
- Refresh is single-flight and bounded.
- Account diagnostics hold only fixed text, status and error codes and exception class names, then are redacted. Never a token, code, header or body.

### On the backend

- Refresh tokens are opaque high-entropy values, stored only as hashes and rotated on every refresh; a rotated token presented again revokes the session.
- Access tokens are short-lived JWTs verified by server time, and every authenticated request re-checks the session, device and user, so revocation is immediate.
- Activation codes are high-entropy, stored only as hashes, shown once, and expire.
- Tokens never appear in URLs or logs; credential-looking query parameters are refused. A test signs in, refreshes, replays and logs out, then asserts that no credential appears in the captured logs.
- Secrets come only from the environment, and the configuration's string form redacts them.
- The backend holds no AI provider credential.
- No custom cryptography: a standard JWT library, and the JDK for hashing and randomness.

## Repository and test hygiene

- Signing material, local machine configuration and environment files are never committed.
- Tests use self-evidently fake credentials.
- Before each public update of this portfolio, its exact contents are scanned for secrets, personal data and local paths.

---

## Not claimed

- No certificate pinning, no root or tamper detection, and no Play Integrity attestation (the app has no Google Play Services dependency).
- Keys are protected at rest, not from the app's own process at runtime, because background runs need them.
- The backend has no rate limiting yet, and it is not deployed.
- No third-party security audit has been performed.
