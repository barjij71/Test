# NOTICE — License Audit & Attribution

This file documents the license status of all dependencies and references used in JARVIS.
Generated as part of Phase 0 (Baseline & Governance).

## JARVIS Direct Dependencies (jarvis/ package.json & Cargo.toml)

| Package | License | Source | Notes |
|---------|---------|--------|-------|
| @tauri-apps/api | MIT | npm | Tauri 2 frontend API |
| @tauri-apps/cli | MIT | npm | Tauri CLI |
| react | MIT | npm | UI framework |
| react-dom | MIT | npm | React DOM renderer |
| typescript | Apache-2.0 | npm | Type checking |
| vite | MIT | npm | Build tool |
| tauri (Rust crate) | MIT/Apache-2.0 | crates.io | Tauri backend |
| tauri-plugin-shell | MIT/Apache-2.0 | crates.io | Shell commands |
| tauri-plugin-fs | MIT/Apache-2.0 | crates.io | Filesystem access |
| tauri-plugin-dialog | MIT/Apache-2.0 | crates.io | Dialogs |
| tauri-plugin-notification | MIT/Apache-2.0 | crates.io | Notifications |
| tauri-plugin-store | MIT/Apache-2.0 | crates.io | Persistent storage |
| tauri-plugin-opener | MIT/Apache-2.0 | crates.io | URL/file opener |
| serde | MIT/Apache-2.0 | crates.io | Serialization |
| serde_json | MIT/Apache-2.0 | crates.io | JSON support |
| tokio | MIT | crates.io | Async runtime |
| reqwest | MIT/Apache-2.0 | crates.io | HTTP client |
| rustls | MIT/Apache-2.0 | crates.io | TLS |
| android-activity | MIT/Apache-2.0 | crates.io | Android entry point |
| jni | MIT/Apache-2.0 | crates.io | JNI bindings |
| okhttp (Kotlin) | Apache-2.0 | Maven | HTTP client |
| kotlinx.coroutines | Apache-2.0 | Maven | Coroutines |
| room (Android) | Apache-2.0 | Maven | SQLite wrapper |
| work-runtime (Android) | Apache-2.0 | Maven | WorkManager |
| flutter_secure_storage equivalent | BSD-3-Clause | pub.dev | Keystore-backed storage |

All direct dependencies use permissive licenses (MIT, Apache-2.0, BSD-3-Clause). No GPL or copyleft licenses detected.

## Baseline Repository (barjij71/jarvis-assistant)

| Component | License | Status |
|-----------|---------|--------|
| Python FastAPI backend | MIT | Permissive |
| Flutter/Capacitor prototypes | BSD-3-Clause | Permissive |
| Computer-control tools | MIT | Permissive |
| SQLite memory layer | MIT | Permissive |
| Provider failover logic | MIT | Permissive |

## Architecture Reference (barjij71/jarvis)

| Component | License | Status |
|-----------|---------|--------|
| Tauri 2 + React + TypeScript | MIT/Apache-2.0 | Permissive |
| Kotlin native plugins | MIT/Apache-2.0 | Permissive |
| Rust backend | MIT/Apache-2.0 | Permissive |
| 6 specialist agents | MIT | Permissive |
| Memory systems (3) | MIT | Permissive |
| Wake word listener | MIT | Permissive |
| PC/TV control | MIT | Permissive |

## Capability Reference — PocketStrike (AbuZar-Ansarii/PocketStrike)

**CRITICAL: PocketStrike is used as a BEHAVIOUR REFERENCE ONLY until its license is confirmed.**

| Feature | License Status | Action |
|---------|----------------|--------|
| MCP client (Dart) | UNKNOWN | Clean-room reimplementation from observed behaviour |
| Hermes memory (USER/MEMORY/SOUL.md) | UNKNOWN | Clean-room reimplementation from observed behaviour |
| Telegram bridge | UNKNOWN | Clean-room reimplementation from observed behaviour |
| Local GGUF inference (llama.cpp) | MIT (llama.cpp) | llama.cpp is MIT; PocketStrike binding UNKNOWN |
| Model downloader | UNKNOWN | Clean-room reimplementation |
| File tools (SAF) | UNKNOWN | Clean-room reimplementation |
| Image generation | UNKNOWN | Clean-room reimplementation |

**llama.cpp**: MIT License — can be vendored as git submodule at pinned commit.

**Models referenced by PocketStrike** (license status per model):
- SmolLM2-360M: Apache-2.0 (HuggingFace)
- Llama-3.2-1B/3B: Llama 3.2 Community License (restricted commercial)
- Phi-3-mini: MIT
- Gemma-2-2B: Gemma License (restricted)
- Qwen2.5-0.5B/1.5B: Apache-2.0

Only models with permissive licenses (Apache-2.0, MIT) will be included in curated presets. Restricted licenses require explicit user opt-in and are documented in modelCatalog.ts.

## Reimplementation Policy (Clean Room)

Since PocketStrike's license is unconfirmed, all PocketStrike capabilities are reimplemented from:
1. Publicly observable behaviour (UI flows, API contracts, file formats)
2. Public documentation and specifications (MCP spec, Telegram Bot API, llama.cpp API)
3. No source code from PocketStrike is copied, translated, or derived.

Each reimplemented module includes a header comment:
```typescript
// Clean-room reimplementation based on observed PocketStrike behaviour.
// PocketStrike source code was NOT consulted. Reference: https://github.com/AbuZar-Ansarii/PocketStrike
// License status of PocketStrike: UNKNOWN — see NOTICE.md
```

## Attribution

Portions of this software are inspired by the PocketStrike project (https://github.com/AbuZar-Ansarii/PocketStrike).
PocketStrike is used as a behavioural reference only; no code is copied.

llama.cpp by Georgi Gerganov et al. (https://github.com/ggerganov/llama.cpp) — MIT License.

Android, Kotlin, and related trademarks are property of Google LLC.
Tauri is a trademark of the Tauri Programme within The Commons Conservancy.
React is a trademark of Meta Platforms, Inc.