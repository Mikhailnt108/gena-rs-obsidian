# NOW

## Current Goal
Implement Gena Chat Completions compatibility adapter from `ROADMAP.md`.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Work is on `main` only; no other git branches used.
- First roadmap slice implemented:
  - new crate `codex-rs/gena-chat-completions-adapter`;
  - Chat Completions mapping moved out of `codex-core/src/client.rs`;
  - `codex-core` keeps thin `WireApi::ChatCompletions` routing/auth/retry/client setup;
  - `WireApi::Responses` path remains untouched.
- Validated:
  - `just fmt`;
  - `cargo check -p gena-chat-completions-adapter`;
  - `cargo check -p codex-core`;
  - `cargo test -p gena-chat-completions-adapter`;
  - `cargo test -p codex-api chat_completions`;
  - `cargo test -p codex-core --test all chat_completions_text_before_tool_call_runs_tool_loop_to_completion`;
  - `just fix -p gena-chat-completions-adapter -p codex-api -p codex-core`.
- `ROADMAP.md` now has a status checklist:
  - phases 0-11 complete;
  - phase 12 open for manual LLMOps validation and optional full workspace test.

## Blockers
- Full workspace `cargo test` not run; requires explicit user approval.

## Next Step
Run manual LLMOps validation for Chat Completions wire API.
