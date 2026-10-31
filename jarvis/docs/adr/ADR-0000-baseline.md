# ADR-0000: Baseline Architecture & Test Baseline

## Status
Accepted

## Context
JARVIS needs a solid baseline before porting PocketStrike capabilities. The owner maintains two existing repositories:
- **barjij71/jarvis-assistant** — Python FastAPI backend + Flutter/Capacitor prototypes (private)
- **barjij71/jarvis** — Tauri 2 + React + TypeScript + Kotlin + Rust (public, working implementation)

The target repository is **barjij71/Test** with app root at `jarvis/`.

## Decision
Adopt **barjij71/jarvis** as the architectural baseline for the following reasons:

1. **Stack alignment**: Tauri 2 + React + TypeScript + Kotlin + Rust matches the mission requirements exactly.
2. **Working implementation**: 542 passing tests, 6 specialist agents, native bridge, wake word, PC/TV control, three memory systems.
3. **Architecture rules documented**: CLAUDE.md exists with 14 rules that govern development.
4. **Production patterns**: Native bridge, dispatch chokepoint, polling-based signals, operator loop — all implemented and tested.

The Python/Flutter baseline (jarvis-assistant) is archived as reference only. Its provider failover logic (Phase 1) will be ported from the `barjij71/codex` branch `feature/embedded-runtime-multiprovider` if accessible.

## Baseline Architecture Summary

### Core Components (from barjij71/jarvis)

| Layer | Technology | Key Files |
|-------|------------|-----------|
| UI | React 18 + TypeScript + Vite | `src/` |
| Backend | Tauri 2 (Rust) | `src-tauri/src/` |
| Android Native | Kotlin | `src-tauri/gen/android/.../MainActivity.kt`, `OperatorCore.kt` |
| Dispatch | Single chokepoint | `src/tools/registry.ts`, `src/tools/dispatch.ts` |
| Native Bridge | Kotlin → Rust #[command] → invoke | `src-tauri/src/commands.rs`, `src-tauri/gen/android/.../plugins/` |
| Memory (Keyword) | store.ts | `src/brain/memory/store.ts` |
| Memory (Semantic) | vectorStore.ts | `src/brain/memory/vectorStore.ts` |
| Memory (Procedural) | proceduralLearning.ts | `src/brain/memory/proceduralLearning.ts` |
| Wake Word | Porcupine (module singleton) | `src/platform/wakeWord.ts` |
| Speech STT | Web Speech API | `src/platform/stt.ts`, `src/platform/webspeech.ts` |
| Speech TTS | Web Speech API (to be replaced per ADR-0002) | `src/platform/tts.ts` |
| Provider Abstraction | brain/providers/ | `src/brain/providers/` |
| Agents | 6 specialists + orchestrator | `src/brain/agents.ts` |
| Sync | phone ↔ PC | `src/brain/sync.ts` |
| Scheduling | Native WorkManager | `src-tauri/gen/android/.../SchedulerWorker.kt` |

### Test Baseline (barjij71/jarvis)

| Test Type | Count | Status | Location |
|-----------|-------|--------|----------|
| Unit | 542 | ✅ Passing | Colocated `*.test.ts` |
| Integration | 23 | ✅ Passing | `tests/integration/` |
| Replay Fixtures | 18 | ✅ Recorded | `src/brain/replay/fixtures.json` |
| E2E (Playwright) | 12 | ✅ Passing | `tests/e2e/` |
| TypeCheck | — | ✅ Clean | `npm run typecheck` |
| Lint (ESLint + clippy) | — | ✅ 0 problems | `npm run lint` |

### Device Matrix

| Device | OS | Architecture | Role |
|--------|-----|--------------|------|
| Samsung SM-E156B | Android 16 | arm64-v8a | Primary phone (owner) |
| Sony BRAVIA | Android TV 14 | arm64-v8a | TV target |
| Pixel 7a (emulator) | Android 15 | x86_64 | CI emulator |
| Desktop (Ubuntu) | Linux | x86_64 | Desktop target |

## Risk Register (Baseline)

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Baseline repo becomes unavailable | Low | High | Fork to barjij71/Test/jarvis-baseline-archive if needed |
| CLAUDE.md rules conflict with mission | Medium | Medium | Document conflicts in ADRs, escalate via decision-needed Issue |
| Test flakiness on CI | Medium | Medium | Quarantine flaky tests, fix in follow-up PR |
| NDK version drift | Medium | Low | Pin NDK version in CI workflow |
| src-tauri/gen/android drift | High | Medium | Commit gen/android; review diff in every PR |

## Migration Path
1. Create initial commit on barjij71/Test:main (done)
2. Create branch jarvis/phase-0-foundation (done)
3. Push all Phase 0 deliverables to branch (in progress)
4. Open PR for review
5. After merge, tag baseline commit as `baseline-2026-10-31`

## Rollback Plan
If baseline adoption proves incorrect:
1. Revert to empty main branch
2. Re-evaluate baseline candidates
3. Document lessons in ADR-0000-amendment

## References
- barjij71/jarvis (architecture reference)
- barjij71/jarvis-assistant (archived reference)
- MISSION.md (this project's requirements)
- CLAUDE.md (architecture rules from baseline)