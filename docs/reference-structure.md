# WhimprFlow Rust reference — full structural record

Captured before `/reference` was deleted, so the design/behavior it embodied isn't
lost. **This is a historical record, not a spec** — see `CLAUDE.md` for the ground
rules (not being replicated, not consulted unless explicitly asked). Where the design
research (`docs/research/`, `SPEC.md`) said one thing and the shipped code did another,
that's called out explicitly below — don't trust the research docs' recommendations
over what's marked "actually shipped."

## What it was

A local-first, cross-platform voice-dictation app cloning the *behavior* of Wispr Flow
(no shared code/assets — built from a teardown of the real app for spec purposes only).
Core loop: hold a hotkey → speak → release → on-device Whisper ASR → LLM cleanup
(local by default, cloud opt-in) → paste at the cursor. Built as a 15-agent research
pass (teardown of the real Wispr Flow 1.6.7 bundle on both platforms, OSS-clone review,
model benchmarking) synthesized into `SPEC.md` / `ARCHITECTURE-DUAL-PLATFORM.md`, then
implemented incrementally (`docs/BUILD-STATUS.md` tracked milestones, though it went
stale before the project's end — the real end state, from source, was further along
than that doc describes: e.g. it says local LLM cleanup is "stubbed," but the shipped
code has a working `whimpr-llm-worker` process wired into both platforms, and Windows
went from "unverified" to end-to-end verified per the README).

## Directory layout

```
reference/
  Cargo.toml, Cargo.lock          workspace root (resolver 2, 8 members incl. src-tauri)
  README.md                        user-facing docs, platform status, build instructions
  LICENSE, .gitignore, dev.sh      dev.sh = `tauri dev` wrapper (the only supported dev entrypoint)
  docs/
    SPEC.md                        733-line synthesized behavior spec (Wispr teardown-derived)
    ARCHITECTURE-DUAL-PLATFORM.md  277-line stack decision doc (supersedes an earlier "native Swift" call)
    BUILD-STATUS.md                milestone tracker (went stale before project end)
    research/                      24 raw research tracks (teardown data, per-topic deep dives)
  crates/
    whimpr-core/      pure Rust, no OS/UI deps — state machine, cleanup, dictionary, settings, stats
    whimpr-ipc/       wire protocol for a shell<->sidecar split — DESIGNED, NEVER WIRED UP (see below)
    whimpr-audio/     cpal mic capture + linear resample to 16kHz
    whimpr-asr/       whisper.cpp (whisper-rs) ASR engine, Metal-accelerated on macOS
    whimpr-cleanup/   OpenAI + Anthropic cloud cleanup providers
    whimpr-llm-worker/ standalone binary: loads a local GGUF model, serves cleanup over stdio JSON
    whimpr-sidecar/   NOT the ipc sidecar — a 180-line macOS-only Fn-key detection smoke test
  src-tauri/          the actual app shell: tray, windows, macOS hotkey hook, Windows hotkey hook,
                       paste, auto-learn, local-LLM-worker supervision, all Tauri commands
  ui/                 React + TypeScript: the Hub (settings/history/dictionary/stats) + the overlay pill
```

## `whimpr-core` — the platform-agnostic brain

Pure Rust, zero OS/UI dependencies, unit-tested (~50 tests total across the workspace).

### State machine (`state/`)

A pure reducer: `StateMachine::step(Input) -> Vec<Action>`, no clock/mic/hook inside it —
time arrives as `Input::Tick { now_ms }`, hotkey transitions as `Input::Trigger(TriggerToken)`.

**`DictationState`** (the phases): `Idle` | `Recording { mode, session, started_ms, warned }`
| `AwaitingLock { tap_up_ms }` (a quick tap discarded, waiting to see if a second tap
locks hands-free) | `Finalizing { session }`.

**`Input`**: `Trigger(TriggerToken)` (`Down{binding,at_ms}` / `Up{binding,at_ms}` /
`Cancel{at_ms}` / `NormalKeyDuringArm`), `Pipeline(PipelineEvent)` (`Committed{session}` /
`Failed{session}`), `Tick{now_ms}`.

**`Action`** (side-effects the shell executes): `StartCapture`, `StopCaptureAndFinalize`,
`DiscardCapture`, `PlayPing`, `RunPipeline`, `ShowBar(BarState)`, `WarnSessionCap`.

**`BarState`** (what the pill shows): `Idle | Recording | Locked | Transcribing | Done |
Cancelled | Error`.

**Behavior implemented and tested:**
- Hold-to-talk: release ≥`HOLD_MIN_MS` (200ms) after press → finalize + paste.
- Quick tap (<200ms) → discard, enter `AwaitingLock`; a second tap within
  `DOUBLE_TAP_MS` (350ms) → locks into hands-free recording (`RecordMode::Locked`,
  ended by a re-press). No second tap within the window → silently returns to idle,
  nothing pasted.
- Esc cancels from any active state (`BarState::Cancelled` shown briefly, then idle).
- `COOLDOWN_MS` (500ms) after a session ends: new presses are ignored, debouncing key
  bounce / immediate re-trigger.
- `SESSION_CAP_MS` (20 min) hard cap, `WARN_AT_MS` (19 min) one-time warning
  (`Action::WarnSessionCap`) before auto-finalizing at the cap.

**`RecordMode`**: `PushToTalk` | `Locked`. **`SessionId(u64)`**: monotonic per-session id.

### Cleanup (`cleanup/`)

- **`levels.rs`** — `CleanupLevel`: `None` (bypasses the model entirely) | `Light`
  (default — filler removal + grammar only, "when unsure, leave as spoken") | `Medium`
  (also tightens wording) | `High` (full rewrite for brevity, still fact-preserving).
  Each level sets a `max_novelty_ratio` ceiling the gates enforce (0.0 / 0.34 / 0.55 /
  0.85) — the fraction of output words that were never spoken.
- **`prompts.rs`** — one `SYSTEM_PROMPT` shared byte-identical by every provider
  (local/OpenAI/Anthropic): explicit ALLOWED-edits list (filler/stutter removal,
  self-correction resolution on cue words like "actually"/"scratch that"/"no wait",
  spoken-punctuation-to-glyph conversion, list/paragraph formatting, number
  normalization, custom-vocabulary substitution) and a NEVER list (never answer
  questions found in the dictation, never add content, never change facts/numbers/
  names/URLs). A 10-pair `FEW_SHOT` set is sent as real user/assistant turns before the
  real transcript — small models follow demonstrations far more reliably than abstract
  rules; each pair isolates one behavior (self-correction, list formatting, "actually"
  as a non-correction, a near-no-op anti-over-editing anchor, etc). A separate, only
  conditionally-used `VERIFIER_PROMPT` asks a model to judge PASS/FAIL on a candidate
  cleanup — designed but not seen wired into the actual runtime path that was read.
  `format_mode_for_app()` maps a frontmost bundle id to a per-medium instruction
  (email/SMS/team-chat/document) appended to the system prompt.
- **`mod.rs`** — the provider trait seam (`CleanupProvider::cleanup(raw, ctx) ->
  anyhow::Result<String>`), `CleanupContext` (level, pre-filtered vocab, app bundle id,
  optional window context), `build_messages()` (assembles system + few-shot + real
  message), and **`post_process()`** — a deterministic belt-and-suspenders pass run on
  every model output regardless of provider: strips a stray markdown code fence, turns
  a sentinel-marker system (`[[NL]]`/`[[NP]]`, inserted pre-model so small models pass
  an opaque marker through reliably instead of "helpfully" rewriting a real newline)
  back into real line breaks, catches any literal "new line"/"new paragraph" cue the
  model missed, and caps runaway blank lines. Correction cue words are deliberately
  *not* touched here — that stays the model's context-sensitive job (a bare regex would
  misfire on "I actually liked it").
- **`gates.rs`** — the deterministic anti-over-editing safety net run on every cleanup
  output before it's trusted: rejects on a banned assistant-style prefix ("here is",
  "certainly", "i'm sorry"...) appearing in the output but not the raw text; a
  must-preserve entity (URL, email, 4+-digit number) present in raw but missing from
  cleaned; >55% character shrinkage; >1.6x growth; or a novelty ratio (fraction of
  output words never spoken) exceeding the level's ceiling. Any gate failure → fall
  back to the raw transcript, never a partial/guessed edit.

### Dictionary (`dictionary/mod.rs`)

`DictionaryStore` — a flat `Vec<DictionaryEntry { correct, mishears, source }>`
(`source` is `Manual` or `Auto`), persisted as JSON, case-insensitive dedup on add.
`prefilter(utterance, max)` selects only entries phonetically close (normalized
Levenshtein ≤0.34) to a spoken token *or adjacent-token bigram* (catches a name split
across two spoken words, e.g. "charge bee" → "ChargeBee") — capped at ~15 entries so the
cleanup prompt only sees what's plausibly relevant to this utterance, not the whole
dictionary.

### Stats (`stats.rs`)

`StatsStore` — an append-only `Vec<SessionRecord>` (timestamp, word count, duration,
char count, the actual text, target app), persisted as JSON, **rewritten in full on
every single dictation** (no rotation/retention — a real scaling concern for long-term
use). `summary(tz_offset_minutes, now_unix)` computes: total words/sessions/speaking
time, avg/best WPM, today's words/WPM, a day streak (consecutive days with activity,
tolerant of "today" being still-empty), last-7-days word counts, and estimated time
saved vs. a 45wpm typing baseline. All day-boundary math takes the tz offset as a
parameter (pure function, no timezone crate dependency) so it's unit-testable and
matches the user's actual local clock.

### Settings (`settings.rs`)

`Settings { cleanup_mode, cleanup_level, openai_model, openai_base_url,
anthropic_model, sound_on_start }`. `CleanupMode`: `Raw | Local (default) | OpenAi |
Anthropic`. `openai_base_url` lets "OpenAI mode" point at any OpenAI-compatible endpoint
(OpenRouter was the documented use case for users without an OpenAI/Anthropic key).
Plain `serde_json` load/save, **not atomic** (no write-temp-then-rename) — a crash
mid-write loses the file silently, same weakness shared by dictionary/stats.

### ASR seam (`asr/mod.rs`)

The trait (`AsrEngine::transcribe(pcm16k) -> Transcript`) and `AsrEngineId` enum list
`FluidAudioAne` (Apple Neural Engine) and `OnnxParakeet` (Windows-primary,
macOS-fallback) as the **researched/planned** engines — see `docs/research/local-asr.md`
and `xplat-asr.md` for that reasoning. **What was actually shipped** (`whimpr-asr` crate)
is `WhisperCpp` via `whisper-rs`, Metal-accelerated on macOS, CPU-only on Windows. This
is the single clearest research-vs-shipped divergence in the codebase.

## Platform-native layers (`src-tauri/src/`)

- **`hotkey.rs`** (macOS) — installs a listen-only CoreGraphics event tap on
  `flagsChanged` for the Fn/Globe key via raw `extern "C"` FFI (no crate wraps this),
  feeds Down/Up into the shared `whimpr-core` state machine, and is the file where
  audio capture, ASR, cleanup dispatch, dictionary, stats, and settings are all wired
  together and held as `OnceLock<Mutex<...>>` globals. Loads the biggest Whisper model
  present (`large-v3-turbo` > `medium.en` > `small.en` > `base.en`, falling back to
  `base.en`). The tap is deliberately created *only after* Accessibility is confirmed
  trusted — macOS fixes a tap's privilege at creation time; one born untrusted never
  upgrades without a relaunch.
- **`win.rs`** (Windows) — a **separate, parallel implementation**, not a thin platform
  shim over the shared state machine. Uses a bare `AtomicBool` for recording state, so
  none of the double-tap-lock, cancel, cooldown, or session-cap behavior exists here —
  only hold-to-record via a `WH_KEYBOARD_LL` hook on Right Ctrl. Text injection is
  clipboard + `SendInput` Ctrl+V. Foreground-app detection via `GetForegroundWindow` +
  `OpenProcess`/`GetModuleBaseNameW`. Selecting "Anthropic" cleanup mode on Windows
  silently falls through to the local worker instead (`_ => run_local()`), a real
  behavioral gap from what the Settings UI implies.
- **`paste.rs`** — macOS: save clipboard → write text → synthesize Cmd+V via
  `CGEventCreateKeyboardEvent`/`CGEventPost` → restore clipboard, gated on
  `AXIsProcessTrusted`. Only saves/restores clipboard *text* (an image/file on the
  clipboard would be destroyed). Fixed 60ms/150ms sleeps around the keystroke, not real
  synchronization.
- **`autolearn.rs`** (macOS only) — after a paste, snapshots the focused field via
  the Accessibility API, waits 7s, re-reads it, and if exactly one word was swapped
  (both ≥3 letters, alphabetic, phonetically close via normalized Levenshtein ≤0.6, the
  replacement Titlecase, neither a common word from a ~90-word stoplist), learns it as
  an auto dictionary entry. Deliberately conservative to avoid poisoning the dictionary
  on ordinary edits. Uses raw AX/CoreFoundation FFI with manual `CFRelease` and an
  `unsafe impl Send` on a raw `AXUIElementRef` held across the 7-second sleep.
- **`local_llm.rs`** — spawns `whimpr-llm-worker` as a child process (stdin/stdout JSON,
  one request per line) specifically so llama.cpp and whisper.cpp never link into the
  same binary (a ggml symbol clash). No timeout on the blocking `read_line`, no
  liveness/respawn logic — a stalled or dead worker silently degrades every subsequent
  cleanup to raw-passthrough with no recovery.
- **`appctx.rs`** — `frontmost_bundle_id()` via `NSWorkspace`, used to snapshot the
  paste target at record-start (before focus can move) for both cleanup formatting and
  stats.
- **`lib.rs`** — the Tauri entrypoint: builds the overlay + Hub windows, tray menu
  (still has "Demo: recording"/"Demo: idle" items), and the full `#[tauri::command]`
  surface (`get_settings`, `set_settings`, `get_stats`, `get_history`, `get_dictionary`,
  `add/remove_dictionary_entry`, `get_status`, `request_microphone/accessibility/
  input_monitoring`, `set_api_key`).

## `whimpr-ipc` — a designed-but-unused architecture

This crate defines a full shell↔sidecar wire protocol (`ShellToSidecar` /
`SidecarToShell` message enums, a length-prefixed JSON codec, a `Hello`/`Ready`
version-handshake, heartbeat-based dead-hook detection with respawn, a 6-rung paste
ladder from `Accessibility` down to `Declined`, secure-input detection, context reads
around the caret) — explicitly so heavy on-device inference in the main process could
never stall the low-level keyboard callback past the OS's hook-removal timeout. **None
of this was actually wired up.** The shipped hotkey hook runs in-process on both
platforms (see `hotkey.rs`/`win.rs` above); `whimpr-sidecar` (the crate name that sounds
like the other half of this protocol) is unrelated — just a minimal standalone Fn-key
detection smoke test. Only `BindingId`/`RecordModeWire` from this crate are actually
used elsewhere. `PasteOptions::default()` sets `concealed: true` (clipboard-history
exclusion) per its own test, but the real paste path never constructs or passes
`PasteOptions` at all — the privacy feature was designed, never connected.

## Cloud + local cleanup providers

- **`whimpr-cleanup`** — `OpenAiProvider` (Chat Completions API, or any
  OpenAI-compatible `base_url`) and `AnthropicProvider` (Messages API), both sending
  the identical `build_messages()` output translated into their own wire envelope.
  15s timeout, no retry logic.
- **`whimpr-llm-worker`** — standalone binary: loads a GGUF model via `llama-cpp-2`
  once (`n_gpu_layers(999)`, i.e. offload everything to Metal), then serves
  `{messages, max_tokens}` → `{text}` JSON requests one per stdin line, using the
  Qwen ChatML template (`<|im_start|>role\ncontent<|im_end|>`) built fresh per request
  with no prompt truncation — a very long transcript could overflow the 4096-token
  context. Greedy sampling only.
- Preferred local models (by presence, largest first):
  `qwen3-4b-instruct-2507-q4_k_m.gguf` > `qwen2.5-1.5b-instruct-q4_k_m.gguf`. Chosen and
  cross-checked against real released-model facts in
  `docs/research/gap-model-verification.md`.

## UI (`ui/`)

React 18 + TypeScript, Vite. Two separate webview roots (Tauri windows): the Hub
(`index.html` → `hub/`) and the overlay pill (`overlay.html` → `overlay/`).

### Overlay — `overlay/FlowBar.tsx`

A single always-on-top transparent window, listens for `whimpr://flowbar/state` and
`whimpr://audio/waveform` Tauri events. Renders a morphing pill: a tiny idle nub, an
expanded recording pill with a cancel button, a canvas-drawn dotted waveform (16 bars,
RMS-driven with an idle shimmer so it never reads as fully static) and a stop button, or
a status-text pill for transcribing/done/cancelled/error. No click-through
(`setIgnoreCursorEvents`) implemented — the pill blocks clicks at its screen position
even while idle.

### Hub — `hub/`

- **`App.tsx`** — top-level router/state. Gates the whole app behind `Onboarding` until
  Accessibility + Microphone are granted; owns `settings`/`status` state and refreshes
  status on mount.
- **`Onboarding.tsx`** — a blocking 3-step permission wizard (Accessibility →
  Microphone → optional Input Monitoring), polls status every 1.2s so steps flip green
  live as macOS grants them, no relaunch needed.
- **`Sidebar.tsx`** — nav: Home / Insights / Dictionary / Snippets / Style / Transforms
  / Scratchpad (the last four routed to a generic `ComingSoon` placeholder, never
  built) / Settings / Help.
- **`Home.tsx`** — a banner, a searchable day-grouped dictation history list (polled
  every 8s), and a stats card (total words, avg WPM, day streak, best WPM + time saved
  once 500 words are crossed — a soft "unlock" gate purely cosmetic, no functional gate
  behind it).
- **`Insights.tsx`** — "Your Usage" tab (WPM gauge as a hand-drawn SVG arc, session
  count, total words, a 7-day bar chart, a streak heatmap — only the most recent
  column of the heatmap carries real data, the other 11 weeks are always zero, i.e.
  historical heatmap depth was never implemented) and a "Your Voice" tab that's a pure
  coming-soon placeholder.
- **`SettingsPane.tsx`** — cleanup engine mode selector (Raw/Local/OpenAI/Anthropic),
  API key fields (writes straight to the OS keychain via `set_api_key`, masked input),
  an OpenAI base-URL + model override pair (for OpenRouter-style use), 4 cleanup-level
  cards, a sound-on-start toggle, and a permissions section mirroring `Onboarding`'s
  three checks with individual re-grant buttons.
- **`DictionaryPane.tsx`** — All/Personal/"Shared with team" tabs (team sharing is a
  permanent coming-soon placeholder, no backend for it exists), search, an inline add
  form (word + comma-separated known-mishears), entries flagged with a ✨ when
  `auto`-sourced.
- **`ui.tsx`** — shared primitives: `Card`, `Dot` (status indicator), `Button` (dark/
  accent/ghost variants), `Segmented` (segmented control), `PageTitle`, and a
  `useStats()` hook polling `get_stats` every 4s.
- **`icons.tsx`, `theme.ts`, `format.ts`, `api.ts`** — an SVG icon set, a light theme
  token map, human-friendly formatters (compact numbers, durations, "≈N news articles"
  word-count framing), and the typed Tauri `invoke` wrapper layer (every function
  catches and falls back to a default so the Hub still renders in a plain browser
  preview without the Tauri shell).
- **`tokens/values.ts`** — the actual design system: a "Deep-Slate / Aqua-Whimpr"
  palette (near-black slate scale + cyan/teal `#22C3B6` accent), pill geometry constants
  (morph duration, idle/recording pill sizes — taken from `SPEC.md`), and the three font
  families (Inter/Geist for UI, Fraunces/Newsreader serif for headings, JetBrains Mono).

## Config

- **`tauri.conf.json`** — `"csp": null` (webview CSP disabled), macOS
  `minimumSystemVersion: "14.0"`, and — a real problem if this were ever built from —
  a **personal Apple Development signing identity + email address committed in plain
  text** (`signingIdentity`). Bundle targets: `app`, `dmg`.
- **`capabilities/default.json`** — minimal Tauri v2 permission set (`core:default`,
  `core:event:default`, `core:window:default`, `core:webview:default`, `core:app:default`)
  for the `main` and `whimpr_bar` windows.
- Workspace `Cargo.toml` — `resolver = "2"`, `edition = "2021"`, `rust-version = "1.82"`,
  release profile: `lto = "thin"`, `codegen-units = 1`, `strip = true`.

## `docs/research/` — topics covered (24 files, not reproduced here)

Wispr Flow teardown facts for both platforms (`app-teardown.md`, `win-teardown.md` — a
real downloaded/extracted/deleted DMG and Squirrel installer analysis), the cleanup
LLM's real architecture and prompting approach (`cleanup-llm-and-api.md`,
`cleanup-prompting.md`, `edit-behavior.md`), a full feature/settings inventory
(`feature-inventory.md`), the Flow Bar UI spec (`ui-flow-bar.md`), hotkey/interaction
behavior on both platforms (`hotkeys-interaction.md`, `win-hotkeys.md`), text-insertion
strategy (`win-insertion.md`), the stack decision and its Windows-specific
reconsideration (`stack-decision.md`, `win-shell.md`, `gap-tauri-vs-electron-overlay.md`),
ASR engine choices and latency budgets (`local-asr.md`, `xplat-asr.md`,
`gap-asr-engine-mac-latency.md`), local-LLM hardware variance
(`win-llm.md`), sidecar-vs-in-process architecture tradeoffs
(`gap-sidecar-vs-inprocess.md` — the analysis behind the `whimpr-ipc` design that never
got wired up), model licensing/benchmark verification (`gap-model-verification.md`),
the auto-learned dictionary design (`gap-auto-learned-dictionary.md`), app
lifecycle/teardown behavior (`app-teardown.md`, `win-teardown.md`), OSS Wispr-alternative
teardowns for legality/reuse review (`oss-clones.md`), and macOS-specific architecture
notes (`macos-architecture.md`). Two files (`mac-gap-critic.md`, `win-gap-critic.md`) are
raw critic-agent JSON output rather than prose.

## Build/run (of the reference project, for the record only)

```bash
cd reference/ui && pnpm install && cd ..
./dev.sh                    # tauri dev — the only supported dev entrypoint; a plain
                             # `cargo build` does NOT bundle the UI, window shows blank
cargo test --workspace      # ~50 unit tests, all in whimpr-core + whimpr-ipc
```
