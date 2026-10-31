# CLAUDE.md — JARVIS Architecture Rules

> Adopted from baseline `barjij71/jarvis` CLAUDE.md. These rules are **non-negotiable** unless changed through an ADR.

## 1. One Dispatch Chokepoint
`tools/registry.ts` advertises; `tools/dispatch.ts` executes. **Never add a second execution path.**

## 2. One Native Bridge
Kotlin → Rust `#[command]` → `invoke("plugin:phone|<name>")`. **Keep JS, Rust, and Kotlin names in sync.**

## 3. Never Claim Success Unless Actual Return Value Says So
Core trustworthiness property. Check the actual return value, not just the absence of an error.

## 4. Native-to-JS Signals: Poll Monotonic Seq Counter
**No Tauri events and no Channels.** Poll a monotonic `seq` counter over `invoke()`. This includes token streaming.

## 5. Desktop-Only Rust Behind `#[cfg(desktop)]`
React uses runtime `IS_MOBILE` branches.

## 6. Wake-Word Listener Is Module-Level Singleton
**Never move it into a React effect.**

## 7. `quota.ts` Leads Desktop `llm/quota.py`
Mirror before assuming parity.

## 8. Three Memory Systems + Hermes
- Keyword: `store.ts`
- Semantic: `vectorStore.ts`
- Procedural: `proceduralLearning.ts`
- **Hermes (Phase 3)**: `hermesStore.ts` — USER.md, MEMORY.md, SOUL.md
**Know which you touch. ADR-0003 defines Hermes relation — no silent duplication.**

## 9. Speech: Android Native TTS Primary (ADR-0002)
**Original "WebView speech only" rule replaced by ADR-0002.**
- Primary: Android `TextToSpeech` via native bridge (`plugin:phone|tts`)
- Fallback: Cloud TTS provider (user-configurable)
- STT: WebView Web Speech API (`platform/stt.ts`, `platform/webspeech.ts`)

## 10. Tests Colocated
`foo.test.ts` beside `foo.ts`. Model behaviour covered by `brain/replay/fixtures.json`.

## 11. Operator Loop Is Native (`OperatorCore.kt`)
WebView pauses ~60s after background. Background work (bot polling, downloads, scheduled jobs) runs natively: WorkManager, foreground service, or Rust. TypeScript starts, stops, and displays state.

## 12. Run `graphify update .` in Each PR's CI Job
Commit any resulting changes in the same PR. If it cannot run in CI, record the blocker in an Issue.

## 13. Search-Grounded Model Estimates Only From Re-rank Button
Never on launch.

## 14. `src-tauri/gen/android` Is Committed
Review its diff in every PR. `npm run lint` = 0 problems.

---

## Phase-Specific Rules (Added via ADRs)

### ADR-0001: Network Policy
- Native HTTP clients only (OkHttp Kotlin, reqwest Rust)
- Certificate pinning for owner-controlled endpoints only
- mTLS optional, owner-hosted only
- Exponential backoff with jitter, honor Retry-After
- Redact all secrets from logs/errors/crash reports

### ADR-0002: Speech
- Android native TTS primary
- Cloud TTS fallback (opt-in)
- WebView STT only

### ADR-0003: Memory Layering
- Hermes = fourth memory system (USER/MEMORY/SOUL.md)
- USER.md ↔ Keyword (store.ts) bidirectional
- MEMORY.md ↔ Keyword + Semantic (vectorStore.ts)
- SOUL.md write-protected (user edits only)
- Sync: CBOR payloads, LWW for USER/MEMORY, manual merge SOUL

---

## Enforcement
- CI runs: `npm run typecheck`, `npm run lint`, `npm test -- --coverage`
- `graphify update .` runs in CI, fails if uncommitted changes
- APK build verifies `src-tauri/gen/android` compiles
- Coverage thresholds: 90% for new modules, 95% diff coverage

---

## Escalation
If a change conflicts with a rule above:
1. **Stop** that change
2. Open Issue labeled `decision-needed` with options and trade-offs
3. Continue with unblocked work
4. Architecture decisions go in `jarvis/docs/adr/`

---

## References
- Baseline: `barjij71/jarvis` (Tauri 2 + React + TypeScript + Kotlin + Rust)
- Mission: `MISSION.md`
- ADRs: `jarvis/docs/adr/`
- Security: `jarvis/docs/SECURITY.md`