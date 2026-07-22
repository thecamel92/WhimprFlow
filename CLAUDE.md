# WhimprFlow

WhimprFlow is a **local-first, cross-platform voice dictation app**: hold a hotkey,
speak, and cleaned-up text lands wherever the cursor is. It re-creates the Wispr-Flow
style dictation workflow from scratch — its own name, its own code, no shared assets.

The core loop:

1. **Hold a hotkey → speak → release.** A floating pill shows idle / recording /
   transcribing / done states.
2. **On-device speech-to-text** via Whisper — no audio ever leaves the machine by default.
3. **Cleanup** — a local LLM (default, offline, private) or an opt-in cloud provider
   (OpenAI/Anthropic) turns the raw transcript into something worth pasting: fillers
   removed, spoken self-corrections resolved ("meet at 2… no wait, 3" → "3"), punctuation
   and paragraph/list breaks applied. Deterministic gates sit in front of the model's
   output and reject edits that lose content, over-delete, or hallucinate — cleanup is an
   enhancement, never a gate the raw transcript can't fall back past.
4. **Paste** into whatever app is frontmost, formatted for the medium where possible.

Supporting features: a personal dictionary (manual + auto-learned corrections), local
usage stats (words dictated, WPM, streaks), a Hub UI for settings/history/dictionary.
API keys, when used, live in the OS keychain — never a plaintext file.

**Status:** early-stage / proof-of-concept quality throughout. Nothing here should be
assumed production-hardened unless verified.

**Full structural record of the Rust reference** (captured before deletion, for when
you explicitly need the detail): `docs/reference-structure.md`.

## `/reference` — Rust prior art, NOT the active project

`/reference` holds a previous **Rust + Tauri** implementation of this same idea (state
machine, ASR/cleanup crates, native hotkey/paste glue, a React Hub UI). It proved the
core loop works end-to-end on macOS.

**Ground rules:**

- **We are not replicating it.** It's prior art to consult for behavior/ideas when
  useful — not a spec, not a source of truth, not something the new build is graded
  against feature-for-feature.
- **Do not read, grep, or explore `/reference` unless I explicitly ask you to.** Don't
  reach into it "just in case" during ordinary work on the active project. If a task
  seems like it needs something from there, ask first rather than pulling it in.
- Its own `README.md`/`Cargo.toml`/`dev.sh` describe how to build *it*, specifically —
  irrelevant to the active project's build/test process.

## Model choices (carried over from prior discussion — re-verify before trusting)

- ASR: Whisper `large-v3-turbo` primary, smaller `*.en` tiers as fallback for low-spec
  machines.
- Cleanup: a mid-size instruct model (Qwen-family was the prior working choice) at Q4
  quantization, sized to available unified memory/VRAM — don't default to the biggest
  model that fits; measure gate-pass-rate and latency against a real eval set before
  picking a default.
- Local cleanup output should never be trusted blindly — keep (or rebuild) the
  deterministic gate layer from `/reference` (`whimpr-core::cleanup::gates`): reject
  edits that lose entities, over-delete, hallucinate, or read as assistant-speak, and
  fall back to the raw transcript. This is what makes a smaller/faster local model a
  defensible default instead of a gamble.

## Lessons from `/reference` worth carrying forward, not repeating

- Keep the state machine (hold/double-tap-lock/cancel/session-cap) as pure, dependency-free,
  unit-tested logic, shared by every platform — `/reference`'s Windows path diverged into
  a separate, untested implementation; don't let that happen again.
- Atomic writes on any persisted JSON (settings/stats/dictionary) — a crash mid-write
  should never silently lose a user's data.
- Real user-visible error states (a model missing, a worker dead, a gate rejection) —
  not console-only logging.
- The hotkey listener needs to live in its own process from day one, importing as little
  as possible — Windows silently and permanently kills a low-level keyboard hook that
  blocks past ~300ms, which is unrecoverable without a relaunch.
