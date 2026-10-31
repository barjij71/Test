# ADR-0003: Memory Layering — Hermes + Three Existing Systems

## Status
Accepted

## Context
JARVIS baseline (barjij71/jarvis) has **three memory systems**:
1. **Keyword Memory** (`store.ts`) — Exact-match fact storage, key-value, persisted to SQLite.
2. **Semantic Memory** (`vectorStore.ts`) — Embedding-based similarity search, FAISS/ONNX, persisted to vector index.
3. **Procedural Memory** (`proceduralLearning.ts`) — Learned routines, user preferences, action sequences.

PocketStrike introduces **Hermes Memory** — three Markdown files with YAML front matter:
- `USER.md` — Preferences, identity, settings
- `MEMORY.md` — Facts, episodes, conversations
- `SOUL.md` — Personality, values, core directives

**Requirement**: Integrate Hermes as a fourth memory system with **no silent duplication** of existing data.

## Decision

### Memory Architecture Map

| System | Scope | Storage | Query | Write Path | TTL |
|--------|-------|---------|-------|------------|-----|
| Keyword (`store.ts`) | Exact facts, settings | SQLite (key-value) | O(1) key lookup | Direct API | ∞ |
| Semantic (`vectorStore.ts`) | Similarity search | FAISS index + metadata | k-NN (cosine) | Embedding pipeline | ∞ |
| Procedural (`proceduralLearning.ts`) | Routines, patterns | SQLite (structured) | Pattern matching | Learning loop | ∞ |
| **Hermes (new)** | **Human-readable, portable, versioned** | **Markdown files (USER/MEMORY/SOUL.md)** | **Full-text + front-matter** | **CRUD + atomic writes** | **∞** |

### Hermes ↔ Existing Systems Mapping

| Hermes File | Primary System | Secondary Index | Sync Behavior |
|-------------|----------------|-----------------|---------------|
| `USER.md` | Keyword (`store.ts`) | — | Bidirectional: USER.md ↔ store.ts (preferences) |
| `MEMORY.md` | Keyword (`store.ts`) + Semantic (`vectorStore.ts`) | Vector index on fact content | Bidirectional: facts → MEMORY.md + vectorStore |
| `SOUL.md` | **Write-protected** (user only) | — | Read-only for agents; user edits only |

### Data Flow Rules (No Silent Duplication)

1. **USER.md ↔ Keyword Memory**
   - `USER.md` is **source of truth** for user preferences.
   - `store.ts` reads `USER.md` on startup (one-time hydration).
   - Changes via Settings UI → write to `USER.md` → notify `store.ts` to update.
   - Agents read preferences from `store.ts` (fast O(1)).
   - **Never**: Agent writes preference → `store.ts` only (must go through `USER.md`).

2. **MEMORY.md ↔ Keyword + Semantic**
   - `MEMORY.md` is **source of truth** for episodic facts.
   - On fact addition: write to `MEMORY.md` → index into `vectorStore.ts` (async).
   - On fact recall: query `vectorStore.ts` for similarity → return `MEMORY.md` references.
   - Keyword exact-match facts: also mirrored to `store.ts` for O(1) lookup.
   - **Migration**: One-time, idempotent import of `store.ts` facts → `MEMORY.md` (Phase 3).

3. **SOUL.md — Write Protection**
   - **Only explicit user edits** modify `SOUL.md`.
   - Digests, agents, tool outputs **never write** to `SOUL.md`.
   - `SOUL.md` loaded at startup → injected into system prompt (read-only).
   - Changes require user confirmation in Settings → Memory → SOUL editor.

### Hermes File Schema

```yaml
# USER.md
---
version: 1
updated: "2026-10-31T10:00:00Z"
preferences:
  theme: "ironman"
  language: "ar"
  tts_voice: "sam_samsung"
  cloud_tts_enabled: false
---
# Markdown body: free-form notes, identity
```

```yaml
# MEMORY.md
---
version: 1
updated: "2026-10-31T10:00:00Z"
facts:
  - id: "fact-uuid-1"
    content: "User prefers Arabic responses."
    source: "user_statement"
    timestamp: "2026-10-30T15:00:00Z"
    tags: ["preference", "language"]
    confidence: 0.95
    version: 1
---
# Markdown body: chronological episodes
```

```yaml
# SOUL.md
---
version: 1
updated: "2026-10-31T10:00:00Z"
personality:
  name: "JARVIS"
  tone: "professional, concise, helpful"
  values: ["privacy", "accuracy", "user_control"]
---
# Markdown body: core directives, behavioral guidelines
```

### Versioning & Atomic Writes
- **Version**: Integer in front matter. Incremented on each write.
- **Atomic write**: Write to `.USER.md.tmp` → `fsync` → `rename` (POSIX atomic replace).
- **History**: Keep last 10 versions as `.USER.md.v{N}` for rollback.
- **Conflict resolution (sync)**: Last-write-wins for USER/MEMORY; manual merge UI for SOUL.

### Sync (phone ↔ PC)
- Extends existing `sync.ts` (CBOR payloads).
- USER.md, MEMORY.md: LWW with vector clock (timestamp + device ID).
- SOUL.md: Manual merge UI (never auto-merge).
- Payload: CBOR only (never JSON-RPC or MCP traffic).

### Hermes Digest (Daily Job)
- Native WorkManager job (daily, ~3 AM).
- Input: Last 24h user messages (redacted: no PII, secrets, file contents).
- Process: Local LLM (on-device) or cloud (with consent) extracts ≤5 candidate facts.
- Output: Review card in UI — user approves/rejects each.
- Approved facts → `MEMORY.md` + `vectorStore.ts`.

## Consequences
- New module: `brain/memory/hermesStore.ts`, `hermesSync.ts`, `hermesDigest.ts`.
- Settings → Memory: Viewer/editor with diff view (React, Glassmorphism).
- One-time migration script for `store.ts` → `MEMORY.md`.
- `vectorStore.ts` indexing pipeline extended for MEMORY.md facts.

## Migration Path
1. Phase 0: This ADR accepted.
2. Phase 3: Implement `hermesStore.ts` (CRUD, versioning, atomic writes).
3. Phase 3: Implement `hermesSync.ts` (extends `sync.ts`).
4. Phase 3: Implement `hermesDigest.ts` (native daily job).
5. Phase 3: Migration script (idempotent, backup first).
6. Phase 3: Settings UI (viewer/editor).

## Rollback Plan
If Hermes causes performance issues or data inconsistency:
1. Disable Hermes digest job (config flag).
2. Fall back to `store.ts` + `vectorStore.ts` as primary.
3. Keep `USER.md`/`MEMORY.md` as export format only.
4. Document in ADR-0003-amendment.

## References
- MISSION.md (Phase 3, Rule 8)
- barjij71/jarvis `src/brain/memory/` (existing three systems)
- PocketStrike Hermes specification (behavioural reference)
- ADR-0001 (sync uses CBOR)