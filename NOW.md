# NOW

## Current Goal
Validate adapter-level Gena Chat Completions verdict with real LLMOps models.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Work is on `main`.
- Chat Completions loop root cause:
  - Qwen3.5 / glm-4.6 can return plain assistant text that promises work but has no `tool_calls`;
  - Codex then sees no pending follow-up and emits false `task_complete`.
- Implemented structural final-answer contract in code commit `9fd8534421`.
- Refined it in code commit `ad1b9c822e`:
  - raw Chat Completions output is classified by `gena-chat-completions-adapter` before stream construction;
  - only `ToolAction` and `FinalAnswer` verdicts become response streams;
  - `ContractViolation` and `MalformedToolCallMarkup` go to retry/diagnostic handling in `codex-core`.
- Contract behavior:
  - with tools available, assistant text without `tool_calls` must be wrapped in `<final_answer>...</final_answer>`;
  - unwrapped no-tool text triggers a contract repair retry instead of `task_complete`;
  - repair retry uses `tool_choice=auto`, allowing either structured `tool_calls` or marked final answer;
  - repeated unwrapped no-tool text becomes a visible diagnostic.
- Rebuilt debug commands:
  - `gena-debug` and `gena-tui-debug` resolve to `$HOME/.local/bin`;
  - both report version `0.130.0`;
  - current binary mtime is `2026-05-15 01:21:42`.

## Blockers
- Full workspace `cargo test` not run; requires explicit user approval.
- Real LLMOps validation may still be affected by intermittent TLS reset to `https://devx-copilot.tech`.

## Next Step
Run manual LLMOps validation on `Qwen3.5-397B-A17B-FP8` and `glm-4.6`.
