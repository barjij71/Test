# JARVIS — Voice-First Android Assistant

This repository contains the JARVIS project, a standalone voice-first Android assistant built with:
- **Tauri 2** + **React** + **TypeScript** (UI)
- **Kotlin** (Android native)
- **Rust** (Tauri backend)

## Project Structure

```
jarvis/           # Main application root
├── src/          # React TypeScript source
├── src-tauri/    # Rust/Kotlin Tauri backend
├── docs/         # Documentation (ADRs, architecture, security)
└── .github/      # CI/CD workflows
```

## Related Projects

- **Baseline**: [barjij71/jarvis-assistant](https://github.com/barjij71/jarvis-assistant) — Existing JARVIS implementation (Python FastAPI + Flutter)
- **Architecture Reference**: [barjij71/jarvis](https://github.com/barjij71/jarvis) — Tauri 2 + React + TypeScript + Kotlin + Rust implementation
- **Capability Reference**: [AbuZar-Ansarii/PocketStrike](https://github.com/AbuZar-Ansarii/PocketStrike) — Flutter/Dart + llama.cpp (behaviour reference only)

## Phases

See [MISSION.md](jarvis/MISSION.md) for the complete porting plan.

| Phase | Focus | Status |
|-------|-------|--------|
| 0 | Baseline, governance, de-risking | 🔄 In Progress |
| 1 | Cloud provider failover | ⏳ Pending |
| 2 | MCP client | ⏳ Pending |
| 3 | Hermes memory | ⏳ Pending |
| 4 | Telegram bridge | ⏳ Pending |
| 5 | Local file tools | ⏳ Pending |
| 6 | Local GGUF inference | ⏳ Pending |
| 7 | Model downloader | ⏳ Pending |
| 8 | UI, RTL, accessibility | ⏳ Pending |
| 9 | Hardening & verification | ⏳ Pending |
| 10 | Release | ⏳ Pending |

## License

MIT License — see [LICENSE](LICENSE) for details.