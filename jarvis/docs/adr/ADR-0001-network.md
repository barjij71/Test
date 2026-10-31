# ADR-0001: Network Policy & Background Execution

## Status
Accepted

## Context
JARVIS requires network access for:
- Cloud LLM providers (OpenAI, Anthropic, Google, etc.)
- MCP servers (Streamable HTTP, SSE)
- Telegram Bot API (long polling, webhooks)
- Model downloads (Hugging Face Hub)
- Sync (phone ↔ PC)

Android background execution constraints:
- WebView pauses ~60s after app backgrounds (OperatorCore.kt)
- WorkManager for deferrable work (API 23+)
- Foreground services for user-visible ongoing work
- Rust can run in background via Tauri mobile plugins

## Decision

### 1. HTTP Clients by Layer
| Use Case | Client | Pinning |
|----------|--------|---------|
| Cloud LLM providers | Rust `reqwest` + `rustls` | No (third-party) |
| MCP servers | Rust `reqwest` + `rustls` | Yes (owner-controlled only) |
| Telegram Bot API | Kotlin `OkHttp` | Yes (owner-controlled webhook) |
| Model downloads | Kotlin `OkHttp` (WorkManager) | Yes (Hugging Face, pinned) |
| Sync (phone↔PC) | Rust `reqwest` + `rustls` | Yes (owner-controlled) |

### 2. TLS & Certificate Pinning
- **Always validate TLS** — no exceptions, no `dangerously_accept_invalid_certs`.
- **Pin only owner-controlled endpoints**: MCP servers, webhook relay, sync server, Hugging Face (pinned commit).
- **Do NOT pin**: Cloud LLM APIs, Telegram Bot API, third-party MCP marketplace servers.
- **Pinning implementation**:
  - Kotlin: `OkHttp` `CertificatePinner` with SHA-256 pins
  - Rust: `rustls` custom `ServerCertVerifier` with pinned SPKI hashes
- **Backup pin**: Each pinned endpoint has 2 pins (current + next). Rotation procedure documented in SECURITY.md.
- **Pin validation failure**: Hard fail — no fallback to unpinned validation.

### 3. Retries & Resilience
- **Exponential backoff with jitter**: `base * 2^attempt + random(0, base)`
- **Base delay**: 1s for API calls, 5s for downloads
- **Max attempts**: 5 for API, 10 for downloads
- **Honor `Retry-After`** (HTTP) and `retry_after` (Telegram)
- **Deduplication**: In-flight request cache keyed by (method, url, body hash). Return cached promise for duplicate calls within 500ms.

### 4. Redaction (Logging & Errors)
- **Never log**: `Authorization` headers, `Bearer` tokens, API keys, bot tokens, Hugging Face tokens, keystore aliases.
- **Redact in errors**: Replace sensitive values with `[REDACTED]` before serialization.
- **Crash reports**: Strip sensitive fields in `panic_hook` and Kotlin `UncaughtExceptionHandler`.

### 5. Background Execution Policy
| Work Type | Mechanism | Trigger |
|-----------|-----------|---------|
| Telegram long polling | Foreground Service | App start / user enables |
| Model downloads | WorkManager + Foreground Service | User taps download |
| MCP polling (if needed) | WorkManager (periodic) | Configured interval |
| Sync (phone↔PC) | WorkManager (periodic) + Foreground Service (large) | Scheduled / manual |
| Scheduled jobs (alarms, routines) | WorkManager (exact) | AlarmManager → WorkManager |
| LLM inference | Foreground Service (if >30s) | User request |

**Rule**: TypeScript (React) starts/stops background work and displays state. Actual work runs in Kotlin (WorkManager/Foreground Service) or Rust (Tauri mobile background).

### 6. mTLS
- Optional, only for owner-hosted servers (MCP, sync, webhook relay).
- Client certs stored in Android Keystore, accessed via JNI from Rust.
- Not used for third-party APIs.

### 7. Bandwidth Awareness
- **Wi-Fi only by default**: Model downloads (>50MB), large file transfers.
- **Mobile data allowed**: API calls, Telegram polling, sync (small), user-overridden downloads.
- **Coalescing**: Batch small requests within 2s windows on mobile.
- **Compression**: CBOR for internal sync payloads; gzip for HTTP APIs.

## Consequences
- Added complexity in HTTP client configuration per use case.
- Pinning requires rotation process (documented in SECURITY.md).
- Background work requires foreground service notifications (user-visible).
- WebView fetch cannot be used for pinned endpoints — must use native clients.

## Migration Path
- Phase 0: Document policy (this ADR).
- Phase 1: Implement provider failover with Rust `reqwest` (no pinning).
- Phase 2: MCP client with pinning for owner servers.
- Phase 4: Telegram with OkHttp pinning for webhook.
- Phase 6/7: Model downloader with OkHttp + WorkManager.

## Rollback Plan
If pinning causes connectivity issues:
1. Disable pinning for affected endpoint via config flag.
2. Investigate root cause (cert rotation, intermediate CA change).
3. Update pins and re-enable.

## References
- MISSION.md (Network Policy section)
- SECURITY.md (threat model, pin rotation procedure)
- barjij71/jarvis `src-tauri/src/network.rs` (existing network utilities)