# Fork roadmap — Wispr-Flow-grade dictation for Claude Code

Fork of [cjpais/Handy](https://github.com/cjpais/Handy). Remotes: `origin` = aferguson14/Handy,
`upstream` = cjpais/Handy. Sync with `git fetch upstream && git merge upstream/main`. Own code
lives in new files with thin hooks, so upstream merges rarely conflict.

Plan reviewed by codex 2026-10-25 (thread `01a174dc-fa27-7d11-b03a-4066be70ef8c`).

**Decided 2026-10-25:** builds and runs on the **laptop** (where the mic and iTerm2 are; the mini is
reached over SSH). Terminal dictation runs the coding-agent formatter **automatically**, not only on
the second shortcut.

## Models: Parakeet and Whisper, switchable

Handy's model selector already switches. Every feature gates on the **loaded model's runtime
capabilities** (`LoadedInfo.supports_vocabulary` / `supports_initial_prompt`, already carried across
the worker boundary in `engine_supervisor/protocol.rs`), never on the model name.

| | Parakeet | Whisper |
|---|---|---|
| `RunOptions.vocabulary` (custom words biasing) | **no** | yes |
| `RunOptions.prompt` (context conditioning) | **no** | yes |
| fuzzy custom-word correction (`text.rs`) | yes | skipped when vocabulary was sent |
| replacement map (new) | yes | yes |
| LLM cleanup | yes | yes |

Source: transcribe.cpp `src/arch/parakeet/capabilities.cpp` sets no vocabulary/prompt feature.
Don't use the whisper-specific `initial_prompt` extension: it can't be combined with
`vocabulary`/`prompt` (`transcribe-cpp/src/family.rs`).

## What upstream already has

- `custom_words`, fuzzy n-gram + Soundex correction (`audio_toolkit/text.rs`)
- LLM cleanup in `actions.rs::process_transcription_output` with Groq, Cerebras, Anthropic, OpenAI,
  OpenRouter, Apple Intelligence (on-device), and an OpenAI-compatible custom URL (Ollama default)
- **Only the separate `transcribe_with_post_process` shortcut post-processes**. `ACTION_MAP`
  hardcodes `transcribe` → false (`actions.rs:959`). `post_process_enabled` only registers the shortcut.

## Stages

**0. Build and use vanilla.** rustup, bun, cmake. Run with Parakeet, check debug log for
`vocabulary=false initial_prompt=false`. Measure LLM latency per provider before choosing.

**1. Replacement map**: own `src-tauri/src/audio_toolkit/replacements.rs` (~250–350 lines with
tests), NOT a rebase of upstream PR #1533 (670 lines, conflicting, built for an older pipeline).
- Hook: `post_process_transcription_text()` in `managers/transcription.rs`, reached through
  `finish_text()`. That covers live, batch fallback, imported files, history retries and headless
  `--transcribe-file`.
- Order: script conversion → **replacements** → fuzzy custom words → filler removal → normalization.
- Semantics: case-insensitive match; boundary = neighbouring char not alphanumeric/underscore (handles
  `Next.js`, `C++`, `.NET`). Output is the replacement's exact casing (do NOT reuse
  `preserve_case_pattern`). Longest match first, single pass (no cascading `A→B→C`). Reject an empty
  `from`; a later duplicate wins. Compile once per settings snapshot.
- Tests: punctuation adjacency, Unicode, apostrophes, `$` in the replacement text, `next` vs `next js` overlap.
- Seed: cloud code→Claude Code, jason→JSON, next js→Next.js, tee mux→tmux, eye term→iTerm2,
  work tree→worktree, verse l→Vercel, plus project names.

**2. Whisper context prompt.** A global `asr_prompt` setting, capability-gated in `run_setup()`
(`transcription.rs:1176`) into `RunOptions.prompt`. Leave it empty for Parakeet so nothing warns. Use
a conservative character cap (no tokenizer available), dedupe against vocabulary, and log truncation.
Echo guard: drop output only if it EXACTLY matches the prompt after normalization. Never use
`contains()`.

**3. LLM deadline.** Configurable total deadline (default ~8 s; Ollama/Apple cold starts exceed 2.5 s).
A timeout behaves like any other `None`, so the raw text is kept. Verify the HTTP request is really
dropped. Test whitespace-only structured output. Port ideas from #2155/#2290 rather than
cherry-picking blind. #1711 (`${custom_words}` in the LLM prompt) is a likely clean pick. #2099
overlaps existing Soundex logic, so assess it first.

**4. DictationContext + PasteOptions** (prerequisite for per-app behaviour)
- Immutable `Arc<DictationContext>` captured in `TranscribeAction::start()` BEFORE `tm.start_stream()`
  (`actions.rs:513`). It holds the frontmost app, resolved ASR prompt, post-process flag, prompt id,
  and paste options. It's passed into `start_stream(ctx)` and `transcribe_recording(.., ctx)`, so live
  and batch fallback agree. Tie it to the recording generation, not a global mutable slot. Late
  completions and rapid restarts are supported today.
- `clipboard.rs:799` rereads global settings at paste. Add `paste_with_options(text, app, PasteOptions)`
  and keep `paste()` as the global-settings wrapper.
- Focus safety: re-read the frontmost app at paste. If it changed since start, still paste but
  force auto-submit OFF.

**5. Minimal app rules**: `app_context.rs` reads `NSWorkspace.frontmostApplication` bundle id
(objc2-app-kit, already a dependency, no extra permission). Non-macOS → None → globals.
Settings is a short ordered list of `{bundle_ids, post_process, prompt_id, asr_prompt, paste_method,
auto_submit}`; first match wins. No profiles UI in v1: edit `settings_store.json` or a small text
field.
- Terminal (iTerm2 `com.googlecode.iterm2`, Terminal, Ghostty, Warp, kitty, WezTerm): Cmd-V paste
  (Direct typing turns `\n` into Enter and submits half a prompt), auto-submit off. Structured
  "coding-agent prompt" template: keep every technical detail, backtick paths/commands/identifiers,
  number multiple asks, no preamble, never answer. Skip the LLM under ~8 words. Bundle id can't tell
  Claude Code from a shell, vim or a password prompt, so the formatter must stay conservative.
- Messages / Slack / Mail: light cleanup, keep tone, no lists or markdown.
- Default: global settings.

**Deferred until daily use asks for it:** profiles CRUD UI, "add current app" button, overlay
profile label, history `profile_id` (needs a history DB migration, and ids go stale on rename),
window-title / Accessibility detection of a Claude Code pane, Windows/Linux window detection.

Retries from history use the CURRENT settings with the entry's old post-process boolean
(`transcription_jobs.rs:307`). Imported files are never post-processed and stay profile-neutral.
