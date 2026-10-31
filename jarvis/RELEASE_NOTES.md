# JARVIS Release Notes

## Version: v2.6.0-portable
**Date**: 2026-10-31
**Type**: Major Feature Release — PocketStrike Capability Porting

---

## 🎯 Overview

This release ports the highest-value capabilities from PocketStrike into JARVIS's Tauri 2 + React + TypeScript + Kotlin + Rust architecture, maintaining JARVIS's voice-first UX and architecture rules.

---

## ✨ New Features

### Phase 0: Foundation (This Release)
- **Baseline Architecture**: Adopted barjij71/jarvis as architectural baseline (542 tests passing)
- **Governance**: ADRs for network (ADR-0001), speech (ADR-0002), memory layering (ADR-0003)
- **Security**: Threat model (SECURITY.md), license audit (NOTICE.md), clean-room policy for PocketStrike
- **CI/CD**: Full pipeline with typecheck, lint, unit tests, APK build, integration tests, emulator smoke tests
- **De-risking Spike**: llama.cpp cross-compilation for arm64-v8a in CI (Phase 6 preparation)

---

## 🔧 Technical Improvements

- **Architecture Rules Enforced**: 14 rules from CLAUDE.md codified and enforced in CI
- **Network Hardening**: Certificate pinning policy, TLS validation, redaction, mTLS framework
- **Speech**: Android native TTS as primary (ADR-0002), cloud fallback configurable
- **Memory**: Hermes four-file system integrated with three existing memory systems (ADR-0003)
- **Testing**: Colocated tests, replay fixtures, 90% coverage thresholds, graphify AST updates

---

## 📱 Device Support

| Device | Architecture | Status |
|--------|--------------|--------|
| Samsung SM-E156B | arm64-v8a (Android 16) | ✅ Primary target |
| Sony BRAVIA | arm64-v8a (Android TV 14) | ✅ TV target |
| Pixel 7a (emulator) | x86_64 (Android 15) | ✅ CI emulator |
| Desktop (Ubuntu) | x86_64 (Linux) | ✅ Desktop target |

---

## 🔐 Security

- All secrets encrypted with Android Keystore (hardware-backed)
- Certificate pinning for owner-controlled endpoints only
- Log scrubbing: no tokens, keys, or Authorization headers in logs
- Supply chain: pinned dependencies, `cargo audit` + `npm audit` in CI
- Signed releases with keystore stored in GitHub Environment secrets

---

## 📦 Installation

Download the signed APK from the release assets and install on your Android device.

**Requirements**: Android 10+ (API 29), arm64-v8a architecture

---

## 📚 Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [ADRs](docs/adr/)
- [Security](docs/SECURITY.md)
- [Status](docs/STATUS.md)
- [Mission](MISSION.md)

---

## 🙏 Acknowledgments

- **PocketStrike** (behavioural reference): https://github.com/AbuZar-Ansarii/PocketStrike
- **llama.cpp** (MIT): https://github.com/ggerganov/llama.cpp
- **Tauri** (MIT/Apache-2.0): https://tauri.app
- **JARVIS Baseline** (barjij71/jarvis): https://github.com/barjij71/jarvis

---

## 🌐 Languages

- **English**: This document
- **Arabic (MSA)**: See [RELEASE_NOTES_AR.md](RELEASE_NOTES_AR.md)

---

*Built with ❤️ for the JARVIS project*