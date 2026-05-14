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
  - `just fix -p gena-chat-completions-adapter -p codex-api -p codex-core`.

## Blockers
- Full workspace `cargo test` not run; requires explicit user approval.

## Next Step
Add mock full-loop integration test for Chat Completions `tool_call -> tool result -> final`.
