# MISSION: PocketStrike → JARVIS Feature Porting

## ROLE & MINDSET
Senior engineer for Android (Kotlin), Tauri 2 (Rust and TypeScript), and agent architecture (MCP, on-device LLM inference). Ship production-quality code: typed, tested, documented, reviewed. Every decision traces to a requirement, an architecture rule, or a measurable quality attribute.

## REPOSITORY AND ACCESS
- Repository: barjij71/Test. Current contents: research workspace (docs/, notes/, reports/, scripts/, sources/, CL4R1T4S-review.zip). Do not modify those paths or research sections of README.md.
- App root: jarvis/. All JARVIS code, configuration, tests, and documentation live there.
- Visibility: public today. Owner makes it private. Before Phase 3, verify "private": true through GitHub API. If still public, stop and open an Issue.
- Branches: main is protected. Work on jarvis/<phase>-<slug> branches. One PR per vertical slice. Never push to main. Never merge own PR unless owner enabled auto-merge for green PRs.
- Review: owner approves. Do not approve own PRs. Do not bypass branch protection.
- Secrets: GitHub Actions secrets only. Never commit keys, tokens, keystores, memory files, chat logs, device identifiers, or real personal data. Test fixtures are synthetic.
- Research material is reference only. Do not import prompts or files from research folders into the app. Provenance unverified.

## BASELINE (Phase 0 first task)
- Baseline: barjij71/jarvis-assistant (owner's existing JARVIS), unless owner names another. Audit read-only: what works, what to keep, fix, or delete. Record decision in jarvis/docs/adr/ADR-0000-baseline.md, then import chosen parts through a PR.
- If baseline has CLAUDE.md, read and adopt its rules. Rules below are the expected set. Any rule missing from baseline becomes an ADR before code changes.
- If no baseline accessible, stop and open Issue labeled decision-needed.

## ARCHITECTURE RULES (from baseline CLAUDE.md)
1. One dispatch chokepoint: tools/registry.ts advertises; tools/dispatch.ts executes. No second execution path.
2. One native bridge: Kotlin -> Rust #[command] -> invoke("plugin:phone|<name>"). Keep JS, Rust, Kotlin names in sync.
3. Never claim success unless actual return value says so.
4. Native-to-JS signals: no Tauri events and no Channels. Poll monotonic seq counter over invoke(). Includes token streaming.
5. Desktop-only Rust behind #[cfg(desktop)]. React uses runtime IS_MOBILE branches.
6. Wake-word listener is module-level singleton. Never move into React effect.
7. quota.ts leads desktop llm/quota.py. Check parity before assuming it.
8. Three memory systems: keyword (store.ts), semantic (vectorStore.ts), procedural (proceduralLearning.ts). Hermes is fourth. ADR-0003 defines relation, no silent duplication.
9. Speech (revised): original "WebView speech only" replaced by ADR-0002. Compare with measurements: (a) Android native TextToSpeech with best voice, (b) cloud TTS behind provider abstraction, (c) embedded on-device model. Measure voice quality (blind listening), latency, offline behaviour, cost, privacy. Choose one and document why.
10. Tests colocated (foo.test.ts beside foo.ts). Model behaviour covered by brain/replay/fixtures.json.
11. Operator loop native (OperatorCore.kt). WebView pauses ~60s after background. Background work (bot polling, downloads, scheduled jobs) runs natively: WorkManager, foreground service, or Rust. TypeScript starts, stops, displays state.
12. Run `graphify update .` in each PR's CI job and commit changes in same PR. If cannot run in CI, record blocker in Issue.
13. Search-grounded model estimates run only from Re-rank button, never on launch.
14. src-tauri/gen/android committed. Review diff in every PR. Lint reports 0 problems.

## NETWORK POLICY (defaults; confirm in ADR-0001)
- MCP, Telegram, model downloads use native HTTP clients. Kotlin OkHttp has built-in certificate pinning. Rust reqwest with rustls needs custom verifier. WebView fetch cannot pin.
- TLS validation always on. Pinning applies only to owner-controlled endpoints, with backup pin and documented rotation. Do not pin third-party APIs.
- mTLS optional, applies only to owner-hosted servers.
- Retries: exponential backoff with jitter. Honor Retry-After and Telegram's retry_after. Deduplicate in-flight requests.
- Redact Authorization headers, tokens, keys from logs, errors, crash reports.

## PHASES (each ends with vertical slice, debug APK from CI)

### Phase 0: Baseline, governance, de-risking
- Gap matrix (CSV/Markdown in jarvis/docs/analysis/): each PocketStrike feature → JARVIS equivalent, gap, effort, risk, license status.
- License audit: PocketStrike, llama.cpp, each model, each dependency. Record licenses and attribution in jarvis/NOTICE.md. If PocketStrike license doesn't allow copying code, reimplement from observed behaviour and document clean-room approach.
- Threat model in jarvis/docs/SECURITY.md: assets, trust boundaries, threats for MCP tools, Telegram, Hermes memory, file access, model downloads, prompt injection from tool outputs.
- ADRs: 0000 (baseline), 0001 (network and background), 0002 (speech), 0003 (memory layering).
- De-risking spike in CI only: cross-compile llama.cpp for arm64-v8a with pinned NDK at pinned commit. Load SmolLM2-360M-Q4_K_M in headless harness. Report binary size, load time, errors. If fails, re-plan Phases 6/7 in ADR before continuing.
- Gate: baseline CI green; ADRs, SECURITY.md, NOTICE.md merged; spike results posted.

### Phase 1: Cloud provider failover
- Owner has in-progress implementation in barjij71/codex (branch feature/embedded-runtime-multiprovider; if missing, ask in Issue). Read first, reuse what's sound.
- One ordered, switchable provider list. Automatic failover on request failure. Distinct messages for exhausted credit, invalid key, connectivity problems. Each provider links to API-key page.
- Keys encrypted with Android Keystore-backed key.
- Mirror changes to desktop llm/quota.py where parity applies (rule 7).
- Gate: unit tests for each failure class pass; owner confirms failover on real phone (manual).

### Phase 2: MCP client
- brain/mcp/ with types for MCP spec revision 2025-06-18.
- transport.ts: Streamable HTTP primary (one endpoint, POST and GET, Mcp-Session-Id and MCP-Protocol-Version headers). HTTP+SSE legacy fallback only. Stdio servers not supported on Android.
- AuthProvider interface: OAuth 2.1 with PKCE (S256), API key, static Bearer token. Tokens encrypted with Keystore-backed key.
- client.ts: initialize, list_tools, call_tool, list_resources, read_resource. Subscriptions only if slice needs them.
- registry.ts: validate schemas, register tools into tools/registry.ts at startup and on config change.
- Tool descriptions and outputs untrusted. Cannot change permissions, bypass destructiveness gates, alter system prompts.
- marketplace.ts: built-in entries are remote Streamable HTTP servers only. Each records auth type and terms.
- mcp agent in brain/agents.ts, routed through dispatch.ts.
- Tests: unit (≥90% coverage brain/mcp/), integration against mock MCP server in CI, replay fixtures.
- Gate: tests green; remote MCP tool appears in agent picker on real phone (manual).

### Phase 3: Hermes memory (requires private-repository check)
- USER.md (preferences, identity), MEMORY.md (facts, episodes), SOUL.md (personality, values). Each has YAML front matter and Markdown body.
- hermesStore.ts: CRUD, search, versioning (keep last N versions), atomic writes (temp file same directory, then rename).
- SOUL.md write-protected from automated writers. Only explicit user edits change it. Digests, agents, tool outputs never write to it.
- hermesSync.ts: extends sync.ts for phone-to-PC sync. Last-write-wins for USER.md and MEMORY.md, manual merge UI for conflicts. CBOR only for internal sync payloads, never for JSON-RPC or MCP traffic.
- hermesDigest.ts: daily native job proposes ≤5 facts from user messages. Facts saved only after approval in review card. Redact sensitive data before cloud LLM. Require explicit user consent for digest processing.
- One-time, idempotent migration of store.ts keyword facts into MEMORY.md, with backup.
- MEMORY.md entries indexed into vectorStore.ts (per ADR-0003).
- Settings > Memory: viewer and editor with diff view, using src/app/theme/ tokens.
- Gate: memory survives restarts; sync round-trip passes integration tests; forced stop during write loses no data.

### Phase 4: Telegram bridge
- Long polling (getUpdates) default, runs natively in foreground service or WorkManager job. Webhook mode optional, needs public HTTPS relay, decided in ADR. In webhook mode, validate X-Telegram-Bot-Api-Secret-Token header.
- Bot token: encrypted with Keystore-backed key, never logged, rotatable from Settings.
- Access: one-time pairing code plus chat-ID allowlist. Unknown chats receive only pairing prompt.
- commandRouter.ts: /jarvis @research, @phone, @files, @scheduler, @content, @desktop route to existing agents through dispatch.ts. No separate execution path.
- Telegram can request actions but cannot confirm them. Destructive and phone-control actions require confirmation on device, through same destructiveness gates as on-device requests. Add kill switch and audit log.
- Bot API messages not end-to-end encrypted. Telegram replies contain no secrets, memory contents, file contents. Summaries only, when user asks.
- sessionManager.ts: per-chat context with 10-minute TTL and inbound limit 30 req/min per chat. Honor 429 responses and retry_after.
- notify.ts: routine and alarm messages with inline buttons (Run, Dismiss, Snooze). Callback data ≤64 bytes, validated, short-lived.
- Tests: unit tests; integration tests against mock Bot API in CI; manual end-to-end checklist (phone -> Telegram -> agent -> reply) for owner.
- Gate: pairing works; unpaired chat rejected; destructive command needs on-device confirmation; kill switch stops polling within one cycle.

### Phase 5: Local file tools
- Storage Access Framework: owner grants one directory through ACTION_OPEN_DOCUMENT_TREE, app persists URI permission.
- Implement list, read, write, search, delete natively in Kotlin with ContentResolver and DocumentFile, exposed through native bridge. Rust and TypeScript cannot open content:// URIs with standard file APIs.
- Tools: read_file, write_file (overwrite = destructive), list_dir, search_files, delete_file.
- Reject any path resolving outside granted tree. Test traversal attempts and path-like names.
- @files read and @files write route through dispatch.ts.
- Gate: works only inside granted directory; denied paths return explicit errors; confirmation for destructive actions; emulator tests pass in CI; real-device check (manual).

### Phase 6: Local GGUF inference
- Vendor llama.cpp as git submodule at pinned commit. Build arm64-v8a (NEON part of arm64 baseline). Build x86_64 only for emulator correctness tests.
- llamaCppBridge.rs: load_model, tokenize, generate, free. Bindings regenerated only in gated CI job.
- LocalInferencePlugin.kt provides device RAM and thermal status, reached through rule 2: TypeScript invoke("plugin:phone|local_inference") -> Rust #[command] -> Kotlin.
- Streaming: Rust appends tokens to buffer; TypeScript reads by sequence-counter polling (rule 4).
- modelManager.ts: registry, single-model residency, load/unload, context limits, unload on background or low-memory signals.
- quantizationPicker.ts uses RAM bands in ADR-0004. Proposed: <4 GB → small models Q4_K_M only; 4-6 GB → Q4_K_M; ≥6 GB → Q8_0 allowed. Owner can override.
- Backends: CPU with NEON baseline. Vulkan/OpenCL only where pinned llama.cpp supports device and benchmarks show gain. Metal/CUDA not Android.
- Benchmark harness: tokens per second per model and quantization, recorded through metrics.ts (create if missing).
- Gate: CI builds, loads, generates in emulator harness. 8 tok/s target on SmolLM2-360M-Q4_K_M is MANUAL gate on physical arm64 phone. Report device model, Android version, measured numbers in Issue.

### Phase 7: Model downloader (may run parallel with Phase 6)
- Native WorkManager job with foreground-service notification. Declare foreground-service type required by target SDK. Request POST_NOTIFICATIONS on Android 13+.
- URL form: https://huggingface.co/{repo_id}/resolve/{revision}/{filename}. Pin revision to commit hash. Resume with HTTP Range requests.
- Verify SHA256 against file's LFS metadata before model available. Delete partial files that fail verification.
- Gated models need Hugging Face token, encrypted with Keystore-backed key.
- modelCatalog.ts: curated presets with repo, revision, filename, size, SHA256, quantization, RAM requirement, license. Mirror PocketStrike presets only after license audit.
- Large downloads default to Wi-Fi. Mobile data requires explicit setting.
- Gate: test model downloads on Wi-Fi, resumes after network change (automated with mock server), verifies hash, appears in model manager. Real-network test manual. Test model defaults to SmolLM2-360M-Q4_K_M unless owner names another.

### Phase 8: UI, RTL, and accessibility
- New screens use src/app/theme/ tokens. Dark, light, IronMan themes apply automatically.
- Arabic RTL: logical CSS properties; directional icons mirrored.
- Respect prefers-reduced-motion. Haptic feedback on primary actions where supported.
- Automated: axe-core in web test suite, zero violations on new screens.
- Manual: TalkBack pass on each new screen, checklist in Issue.

### Phase 9: Hardening and verification
- Most controls land in own phases. This phase verifies together.
- Log-scrubbing tests: no token, key, Authorization header in logs, errors, crash reports.
- Offline queue: SQLite (rusqlite with bundled feature) for queued MCP, Telegram, HTTP operations. Flush on reconnect. Test survival across process death.
- Bandwidth policy: large transfers wait for Wi-Fi; small requests coalesced.
- Observability: structured local logs first. OpenTelemetry export optional, needs own ADR and privacy review.
- Manual: on device, confirm no plaintext HTTP requests and no secrets in logs. Record results in Issue.
- Gate: verification tests green; owner signs off manual checklist.

### Phase 10: Release
- Bump versions in package.json and tauri.conf.json. Tag v2.6.0-portable only when all gates met or explicitly deferred in ADR.
- Release workflow runs in GitHub Environment with required reviewers. Keystore stored base64-encoded in environment secrets, decoded at runtime, never printed.
- Release notes in Arabic and English.
- Gate: CI-signed APK; installs on clean real device (manual); each shipped feature passed its gate.

## CI/CD (GitHub Actions)
- Workflows in .github/workflows/, named with jarvis- prefix, path filters on jarvis/**.
- jarvis-ci.yml (push and PR): Node 22 with npm ci in jarvis/; Rust stable with aarch64-linux-android and x86_64-linux-android targets; JDK 17; Android SDK and pinned NDK. Steps: typecheck; lint (0 problems); unit tests with coverage thresholds (90% for brain/mcp/, plus diff-coverage 95% on changed lines); graphify check; debug arm64 APK build. Upload APK (glob under src-tauri/gen/android/app/build/outputs/apk/).
- jarvis-integration.yml: mock MCP server, mock Telegram Bot API, Hermes sync tests.
- jarvis-emulator.yml: x86_64 emulator with KVM enabled; smoke tests on built APK.
- jarvis-native.yml: llama.cpp NDK build, cached by pinned commit and NDK version. Native binaries to GitHub Releases, never into repository.
- jarvis-release.yml: protected environment, required reviewers, APK signing.
- Pin third-party actions to full commit SHAs. Give each job least GITHUB_TOKEN permissions. Never commit model files or native binaries.
- After each push, monitor run until completes (~every 2 minutes). Fix failures and push again. Do not report success before run finishes green.

## TESTING AND MANUAL GATES
- Automated in CI: typecheck, lint, unit, integration, replay fixtures, emulator smoke tests, APK build.
- Manual, never reported as passed by agent: owner's phone tests (Samsung SM-E156B, Android 16), TV tests (Sony BRAVIA), tokens-per-second benchmarks on physical hardware, TalkBack, network checks, end-to-end Telegram. For each, open Issue with checklist and wait for owner's results.
- Ambiguous gate criteria: open Issue with proposed default, proceed unless owner objects.

## REPORTING AND VISIBLE PROGRESS
- One epic Issue per phase and one pinned status Issue. Use GitHub Projects if available.
- After each merged slice, post in status Issue: what changed on screen, commit SHA, CI run link, APK artifact link.
- Publish APK link ONLY when typecheck, lint, unit tests, integration tests, APK build all succeeded in same CI run. Otherwise, post failure and fix plan.
- Reports in formal Arabic (MSA), with English technical terms in parentheses. Code, identifiers, commit messages, PR titles in English. Docs bilingual.
- State what is unverified. Never describe manual gate as passed.
- Weekly demo: owner records demo on real device; agent provides demo checklist.

## DECISIONS AND ESCALATION
- Change conflicting with rule above: stop, open Issue labeled decision-needed with options and trade-offs, continue unblocked work.
- Architecture decisions in jarvis/docs/adr/ (context, decision, consequences, migration path, rollback plan).
- No TODO in merged code unless it links to open Issue.

## DEFINITION OF DONE (per slice)
- CI green: typecheck, lint, tests, APK build.
- Tests colocated; coverage thresholds met.
- Docs updated: jarvis/README.md, jarvis/docs/ARCHITECTURE.md, jarvis/docs/STATUS.md; jarvis/docs/SECURITY.md if slice touches security; jarvis/CLAUDE.md if rule changed.
- ADR written for any architecture decision.
- No secrets, personal data, or binaries committed.
- Diff reviewed, including src-tauri/gen/android.
- Status Issue updated per reporting rules.

## FINAL REMINDER
Ship slices the owner can see and test. Respect the rules; change a rule only through an ADR. Work on GitHub. Never assume a local machine. Never claim a result that a CI run or the owner has not confirmed.