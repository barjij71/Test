# JARVIS Architecture Documentation

Version: 1.0 (Phase 0 Baseline)
Last Updated: 2026-10-31

## Overview

JARVIS is a standalone, voice-first Android assistant built with:
- **Tauri 2** — Rust backend, WebView frontend
- **React 18 + TypeScript** — UI layer
- **Kotlin** — Android native plugins (phone, notifications, SAF, TTS, etc.)
- **Rust** — Core logic, MCP client, local inference, sync

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ANDROID APP                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐    ┌──────────────────────────────────┐  │
│  │   React UI       │    │   Tauri Rust Core                │  │
│  │   (WebView)      │◄───►│   (src-tauri/src/)               │  │
│  └────────┬─────────┘    └──────────────┬───────────────────┘  │
│           │                               │                    │
│           │ invoke()                      │ #[command]         │
│           ▼                               ▼                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              KOTLIN NATIVE PLUGINS                        │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │  │
│  │  │ Phone   │ │ TTS     │ │ SAF     │ │ WorkMgr │  ...   │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │ Cloud LLMs  │      │ MCP Servers │      │ Telegram    │
    │ (OpenAI,    │      │ (Streamable │      │ Bot API     │
    │  Anthropic) │      │  HTTP/SSE)  │      │ (Long Poll) │
    └─────────────┘      └─────────────┘      └─────────────┘
```

## Core Modules

### 1. Dispatch System (`src/tools/`)
- **registry.ts** — Tool definitions, schemas, permissions
- **dispatch.ts** — Single execution chokepoint, validation, routing

### 2. Native Bridge (`src-tauri/src/commands.rs`)
- Rust `#[command]` functions exposed to TypeScript via `invoke()`
- Kotlin plugins registered as `plugin:phone|<name>`
- Sequence counter polling for native→JS signals

### 3. Memory Systems (`src/brain/memory/`)
| System | File | Purpose |
|--------|------|---------|
| Keyword | `store.ts` | Exact-match facts, settings (SQLite) |
| Semantic | `vectorStore.ts` | Embedding similarity search (FAISS) |
| Procedural | `proceduralLearning.ts` | Learned routines, patterns |
| **Hermes** | `hermesStore.ts` | **USER.md, MEMORY.md, SOUL.md** (Phase 3) |

### 4. Provider Abstraction (`src/brain/providers/`)
- Unified interface for cloud LLMs (OpenAI, Anthropic, Google, etc.)
- Failover, quota tracking, key management (Keystore-encrypted)

### 5. Agents (`src/brain/agents.ts`)
- 6 Specialist agents: Research, Phone, Files, Scheduler, Content, Desktop
- Orchestrator routes to specialists
- MCP agent (Phase 2) routes to MCP tools

### 6. Sync (`src/brain/sync.ts`)
- Phone ↔ PC memory sync
- CBOR payloads, LWW conflict resolution
- Extends for Hermes in Phase 3

### 7. Platform (`src/platform/`)
- `wakeWord.ts` — Porcupine singleton
- `stt.ts` / `webspeech.ts` — Web Speech API STT
- `tts.ts` — **Android native TTS** (ADR-0002) via native bridge

## Native Plugins (Kotlin)

| Plugin | Purpose | Bridge Command |
|--------|---------|----------------|
| PhonePlugin | Calls, SMS, contacts | `plugin:phone|phone_*` |
| TTSPlugin | Android TextToSpeech | `plugin:phone|tts` |
| SAFPlugin | Storage Access Framework | `plugin:phone|saf_*` |
| WorkManagerPlugin | Background jobs | `plugin:phone|work_*` |
| NotificationPlugin | Foreground service, notifications | `plugin:phone|notify_*` |
| LocalInferencePlugin | Device RAM, thermal, llama.cpp | `plugin:phone|local_inference` |
| OperatorCore | Background operator loop | Internal |

## Data Flow

### User Voice Command
```
Voice → STT (WebView) → Text → Dispatch → Agent → Tool → Result → TTS (Native) → Voice
```

### MCP Tool Call
```
Agent → Dispatch → MCP Client → Streamable HTTP → MCP Server → Tool Result → Dispatch → Agent
```

### Background Work
```
TypeScript → invoke("plugin:phone|work_schedule") → Kotlin WorkManager → Rust/Kotlin work → Notification → TypeScript polls seq
```

## Security Architecture

- **Keystore**: Hardware-backed encryption for all secrets
- **Pinning**: Owner-controlled endpoints only (MCP, sync, webhook, HF)
- **Redaction**: Middleware strips secrets from logs, errors, crashes
- **SAF**: User-granted directory only, Kotlin DocumentFile enforcement

## Build & Deploy

### CI Pipeline (`.github/workflows/`)
- `jarvis-ci.yml` — Typecheck, lint, unit tests, graphify, APK build
- `jarvis-integration.yml` — Mock MCP, Telegram, Hermes sync tests
- `jarvis-emulator.yml` — x86_64 emulator smoke tests
- `jarvis-native.yml` — llama.cpp NDK build (cached)
- `jarvis-release.yml` — Signed APK, GitHub Release

### Artifacts
- Debug APK: `jarvis-ci` → `app-debug-arm64.apk`
- Release APK: `jarvis-release` → GitHub Release assets
- Native libs: `jarvis-native` → GitHub Releases (llama.cpp)

## References
- [MISSION.md](../MISSION.md)
- [ADRs](../docs/adr/)
- [SECURITY.md](../docs/SECURITY.md)
- [STATUS.md](../docs/STATUS.md)
- Baseline: https://github.com/barjij71/jarvis