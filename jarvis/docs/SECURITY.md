# SECURITY.md — Threat Model & Security Controls

Version: 1.0
Date: 2026-10-31
Status: Phase 0 Baseline

## 1. Assets

| Asset | Description | Sensitivity |
|-------|-------------|-------------|
| User preferences (USER.md) | Identity, theme, language, TTS settings | Medium |
| User facts & episodes (MEMORY.md) | Personal info, conversations, habits | High |
| Personality core (SOUL.md) | Behavioral directives, values | Medium |
| API keys (LLM providers) | OpenAI, Anthropic, Google, etc. | Critical |
| Telegram bot token | Bot authentication | Critical |
| Hugging Face token | Gated model access | High |
| MCP server credentials | OAuth tokens, API keys | High |
| Keystore master key | Encrypts all above at rest | Critical |
| Voice recordings (STT) | Transient audio buffers | High |
| File system access (SAF) | User-granted directory contents | High |
| Downloaded models | GGUF files (100MB-4GB) | Medium |
| Sync data (phone↔PC) | Encrypted memory payloads | High |
| Crash logs / diagnostics | May contain redacted secrets | Low-Medium |

## 2. Trust Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                      USER DEVICE                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  React UI    │  │  Rust Core   │  │  Kotlin Native   │  │
│  │  (WebView)   │◄─►│  (Tauri)     │◄─►│  (Plugins)       │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                     │            │
│         ▼                 ▼                     ▼            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ANDROID SANDBOX (UID)                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │   │
│  │  │ KeyStore    │  │ File System │  │ Network     │   │   │
│  │  │ (Hardware)  │  │ (App dir +  │  │ (OkHttp/    │   │   │
│  │  │             │  │  SAF tree)  │  │  reqwest)   │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                 │                     │            │
│         ▼                 ▼                     ▼            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Cloud LLMs   │  │ MCP Servers  │  │ Telegram Bot API │  │
│  │ (API keys)   │  │ (Owner ctrl) │  │ (Bot token)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│         │                 │                     │            │
│         ▼                 ▼                     ▼            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Hugging Face │  │ Sync Peer    │  │ Webhook Relay    │  │
│  │ (HF token)   │  │ (Owner ctrl) │  │ (Owner ctrl)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Trust Assumptions**:
- Android Keystore (hardware-backed) is root of trust for encryption keys.
- Owner-controlled servers (MCP, sync, webhook) are trusted for pinning.
- Cloud LLM providers are semi-trusted: API keys protected, but prompts/responses visible to provider.
- Telegram Bot API: not E2E encrypted; bot token is secret; messages visible to Telegram.
- User grants SAF directory — app cannot escape that tree.

## 3. Threats by Capability

### 3.1 Cloud LLM Providers (Phase 1)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| API key exfiltration | Medium | Critical | Keystore encryption; never in logs; redacted in errors |
| Prompt injection via tool output | Medium | High | Tool outputs untrusted; sanitized before prompt injection; no system prompt override |
| Provider failover to malicious endpoint | Low | Critical | Pinned provider list; HTTPS only; cert validation; no user-added providers in Phase 1 |
| Credit exhaustion / billing attack | Medium | Medium | Quota tracking (quota.ts); per-request limits; user alerts |

### 3.2 MCP Client (Phase 2)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Malicious tool description/schema | Medium | High | Schemas validated; tools cannot change permissions or bypass gates |
| MCP server compromise | Low | High | Certificate pinning for owner servers; mTLS optional; auth tokens encrypted |
| Tool output injection | Medium | High | Outputs treated as untrusted; length limits; no code execution |
| Session hijack (Mcp-Session-Id) | Low | Medium | Secure random session IDs; HTTPS only; short TTL |

### 3.3 Hermes Memory (Phase 3)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| MEMORY.md exfiltration | Medium | High | File permissions (app-only); encrypted at rest via Keystore; no cloud sync without consent |
| SOUL.md tampering | Low | High | Write-protected; user edits only; manual merge on sync |
| Digest PII leakage to cloud LLM | Medium | High | Redaction pipeline before cloud; local LLM option; explicit consent required |
| Sync conflict data loss | Medium | Medium | LWW with vector clock; version history (10 versions); manual merge UI |

### 3.4 Telegram Bridge (Phase 4)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Bot token theft | Low | Critical | Keystore encryption; never logged; rotatable in Settings |
| Unauthorized chat access | Medium | High | Pairing code + chat-ID allowlist; unknown chats get pairing prompt only |
| Command injection via Telegram | Medium | High | Destructive actions require on-device confirmation; kill switch |
| Message replay / MITM | Low | Medium | HTTPS only; webhook secret validation; update_id monotonic |
| 429 / rate limit abuse | Medium | Low | Per-chat 30 req/min; honor retry_after; exponential backoff |

### 3.5 Local File Tools (Phase 5)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Path traversal escape | Medium | High | Kotlin DocumentFile API resolves within SAF tree; reject any resolution outside |
| Malicious file write | Medium | High | Overwrite = destructive gate; confirmation UI; no execute permission |
| SAF permission revocation | Low | Medium | Persist URI permission; handle gracefully; re-prompt user |

### 3.6 Local GGUF Inference (Phase 6)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Model file tampering | Low | High | SHA256 verified on load; models in app-private dir |
| Memory exhaustion (OOM) | Medium | High | Single-model residency; unload on background/low-memory; context limits |
| Thermal throttle / battery drain | Medium | Medium | Thermal status from Kotlin; pause inference if critical; benchmark harness |
| Prompt injection via local model | Medium | High | Same as cloud: tool outputs untrusted; no system prompt override |

### 3.7 Model Downloader (Phase 7)
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Malicious model (pickle exploit) | Low | Critical | Only GGUF (safe format); no PyTorch/TensorFlow; verify SHA256 from LFS metadata |
| MITM on download | Low | High | HTTPS only; pin Hugging Face cert (backup pin); verify SHA256 post-download |
| Partial file execution | Low | High | Verify hash before marking available; delete partial on failure |
| Mobile data overage | Medium | Low | Wi-Fi default for large; user opt-in for mobile; bandwidth scheduler |

### 3.8 Cross-Cutting
| Threat | Likelihood | Impact | Mitigation |
|--------|------------|--------|------------|
| Keystore key extraction | Low | Critical | Hardware-backed Keystore (StrongBox where available); no software fallback for master key |
| Log injection / secret leakage | Medium | High | Structured logging; redaction middleware; crash report scrubbing |
| Supply chain (dependency) | Low | Critical | Pin dependencies; `cargo audit` / `npm audit` in CI; signed releases |
| Unsigned APK installation | Low | Critical | Release workflow signs with keystore; verify signature on install |
| Background work persistence | Medium | Medium | WorkManager survives reboot; Foreground Service shows notification |

## 4. Security Controls Summary

| Control | Implementation | Phase |
|---------|----------------|-------|
| Encryption at rest | Android Keystore (AES-256-GCM) for all secrets | 0 |
| Certificate pinning | OkHttp (Kotlin), rustls custom verifier (Rust) | 1,2,4,7 |
| mTLS | rustls + client certs from Keystore (optional) | 2,3 |
| Redaction | Middleware in Rust/Kotlin/TS; panic hook; uncaught handler | 0 |
| Input validation | Schema validation (MCP, Telegram, file paths); length limits | 2,4,5 |
| Destructiveness gates | Confirmation UI for delete, overwrite, phone control, spend | 2,4,5 |
| Kill switch | Telegram polling stop; MCP disconnect; inference cancel | 2,4,6 |
| Audit log | Structured events (no secrets); local + optional export | 9 |
| Log scrubbing tests | CI tests: grep for tokens/keys in logs = 0 matches | 9 |
| Offline queue encryption | SQLite + SQLCipher (or rusqlite + Keystore key) | 9 |

## 5. Pin Rotation Procedure

For each pinned endpoint (MCP server, webhook relay, sync peer, Hugging Face):

1. **Generate new certificate** (30 days before expiry).
2. **Add as backup pin** in config (deploy to both phone and server).
3. **Wait for rollout** (all clients have new pin).
4. **Rotate primary** → make backup the new primary.
5. **Remove old pin** after 30 days.

Emergency rotation: If private key compromised, immediately deploy new cert with new pin, revoke old cert.

## 6. Incident Response

| Incident | Detection | Response |
|----------|-----------|----------|
| API key leaked | Log scrubbing test fails / user reports | Rotate key immediately; revoke old; audit access |
| Bot token leaked | Telegram security alert / unusual activity | Revoke token via BotFather; issue new; update Keystore |
| Model hash mismatch | Downloader verification fails | Delete file; alert user; investigate supply chain |
| Path traversal attempt | File tool returns error | Log attempt; rate-limit; alert if repeated |
| Prompt injection detected | Tool output sanitizer flags | Block output; log; analyze pattern |

## 7. Compliance & Privacy

- **No telemetry** by default. Optional crash reports (opt-in) are scrubbed.
- **GDPR/PDPL**: User data (MEMORY.md, USER.md) exportable and deletable via Settings.
- **Data minimization**: Only necessary data sent to cloud LLMs. Redaction pipeline for digests.
- **Children**: Not directed at children under 13. No COPPA concerns.

## 8. Security Testing (CI)

| Test | Tool | Gate |
|------|------|------|
| Dependency audit | `cargo audit`, `npm audit` | 0 critical/high |
| Secret scanning | `gitleaks`, `trufflehog` | 0 findings |
| Log scrubbing | Custom grep test | 0 matches for token patterns |
| Pin validation | Integration test with rotated cert | Pass |
| Path traversal | Fuzzing test (1000 attempts) | 0 escapes |
| OOM handling | Emulator stress test | No crash, graceful unload |

## 9. References

- ADR-0001 (Network Policy)
- ADR-0002 (Speech — no cloud by default)
- ADR-0003 (Memory — SOUL.md write protection)
- MISSION.md (Network Policy, Rules 3, 4, 9)
- OWASP Mobile Top 10
- Android Security Best Practices