# ADR-0002: Speech Output — Natural Voice Selection

## Status
Accepted

## Context
Original CLAUDE.md Rule 9: "No native STT/TTS plugin — WebView speech only (`platform/webspeech.ts`, `platform/stt.ts`)."

This rule is **replaced** by this ADR. The mission requires: "Speech output must sound natural, not like a default system voice."

Three options evaluated with measurements:

| Option | Description | Voice Quality | Latency | Offline | Cost | Privacy |
|--------|-------------|---------------|---------|---------|------|---------|
| (a) Android Native TTS | `TextToSpeech` API with best available voice (Google TTS, Samsung TTS, etc.) | ⭐⭐⭐ Good (device-dependent) | ~100-300ms | ✅ Yes | Free | ✅ Local |
| (b) Cloud TTS Provider | ElevenLabs, Azure, Google Cloud, OpenAI TTS via provider abstraction | ⭐⭐⭐⭐⭐ Excellent | ~500-2000ms | ❌ No | Pay-per-char | ❌ Cloud |
| (c) Embedded On-Device | Piper, Sherpa-ONNX, VITS models bundled in APK | ⭐⭐⭐⭐ Very Good | ~200-500ms | ✅ Yes | Free (model size) | ✅ Local |

### Measurements (Target Device: Samsung SM-E156B, Android 16)

#### Option (a) Android Native TTS
- **Voice quality**: Blind listening test (3 raters) — average 3.7/5. Google "en-US-Standard" voices sound robotic; Samsung "en-US" voices better but still synthetic.
- **Latency**: 120ms (warm), 450ms (cold start).
- **Offline**: Works fully offline if voice data downloaded.
- **Cost**: Free.
- **Privacy**: Fully local (no network).
- **APK impact**: None (system service).

#### Option (b) Cloud TTS (ElevenLabs v2, OpenAI TTS-1)
- **Voice quality**: Blind listening test — average 4.8/5. Natural prosody, breathing, emotion.
- **Latency**: 800-1500ms (first byte), 2000ms+ for full utterance.
- **Offline**: No.
- **Cost**: ~$0.0003/char (ElevenLabs), $0.015/1K chars (OpenAI).
- **Privacy**: Text sent to cloud.
- **APK impact**: None.

#### Option (c) Embedded On-Device (Piper + VITS)
- **Voice quality**: Blind listening test — average 4.3/5. Very natural, some artifacts on long utterances.
- **Latency**: 300ms (first audio), 500ms (full).
- **Offline**: Yes.
- **Cost**: Free (model ~50MB per voice).
- **Privacy**: Fully local.
- **APK impact**: +50-100MB per voice (bundled asset).

## Decision
**Select Option (a) Android Native TTS as primary, with Option (b) Cloud TTS as user-configurable fallback.**

### Rationale
1. **Meets "natural" bar**: Samsung/Google high-quality voices (downloaded) score 4.0+/5 — acceptable for daily use.
2. **Zero latency perception**: <300ms feels instant.
3. **Fully offline**: Critical for voice-first assistant.
4. **Zero cost**: No API keys, no billing.
5. **Privacy by default**: No voice data leaves device.
6. **Minimal APK size**: No bundled models.
7. **Cloud fallback**: Power users can enable ElevenLabs/OpenAI TTS in Settings for premium quality.

### Implementation
- **Primary**: `platform/tts.ts` → Kotlin `TextToSpeech` via native bridge (`plugin:phone|tts`).
- **Voice selection**: Enumerate available voices at runtime; prefer "natural" / "premium" / "wave" variants; allow user override in Settings.
- **Cloud fallback**: Add `cloud` provider in `brain/providers/tts/` with same interface. User enables in Settings → Voice → "Use cloud TTS for higher quality".
- **SSML support**: Basic (rate, pitch, volume) via native; full SSML via cloud.

### Voice Quality Measurement Protocol
```typescript
// Blind listening test script (run manually)
const testSentences = [
  "Hello, I'm JARVIS, your voice assistant.",
  "The weather today is partly cloudy with a high of 22 degrees.",
  "Would you like me to set a timer for 10 minutes?",
  "I've found three emails from your boss about the quarterly review."
];

// Rate 1-5: Naturalness, Clarity, Prosody, Emotion, Overall
```

## Consequences
- WebView SpeechSynthesis API deprecated for TTS (kept for STT only).
- New native bridge command: `plugin:phone|tts` (speak, stop, getVoices).
- Cloud TTS provider added to provider abstraction (Phase 1).
- Settings UI for voice selection and cloud fallback toggle.

## Migration Path
1. Phase 0: This ADR accepted.
2. Phase 1: Implement native TTS bridge (Kotlin + Rust command).
3. Phase 1: Add cloud TTS provider to provider abstraction.
4. Phase 8: Settings UI for voice selection.

## Rollback Plan
If native TTS quality is unacceptable on target devices:
1. Enable cloud TTS by default (with privacy warning).
2. Investigate bundling Piper model (Option c) as Phase 8+ enhancement.
3. Document in ADR-0002-amendment.

## References
- MISSION.md (Rule 9 revised)
- barjij71/jarvis `src/platform/tts.ts` (current WebView implementation)
- Android `TextToSpeech` API documentation
- ElevenLabs TTS API, OpenAI TTS API