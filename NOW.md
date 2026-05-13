# NOW

## Current Goal
Post-merge release path for Gena `0.130.0`.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Code branch: `main`, synced with `origin/main`, head `2734ce9d5f`.
- Upstream `rust-v0.130.0` is merged and pushed to `main`.
- Full workspace gate passed on 2026-05-13:
  - `RUST_MIN_STACK=16777216 CARGO_BUILD_JOBS=2 cargo test --workspace --all-targets --no-fail-fast -- --test-threads=1`
- Debug build exists:
  - `codex-rs/target/debug/gena-debug` -> `gena 0.130.0`
  - `codex-rs/target/debug/gena-tui-debug` -> `gena-tui 0.130.0`
- Debug install refreshed in `/opt/homebrew/bin`:
  - `/opt/homebrew/bin/gena-debug` -> `gena 0.130.0`
  - `/opt/homebrew/bin/gena` wrapper points to refreshed debug `gena.bin` -> `gena 0.130.0`
  - `/opt/homebrew/bin/gena-tui` -> `gena-tui 0.130.0`
- Shell `gena` without absolute path currently resolves first to `/Users/mntabunkov/.nvm/versions/node/v22.22.2/bin/gena`, which reports old `0.128.0`; use `gena-debug` or `/opt/homebrew/bin/gena` for the debug smoke path.
- Automated Gena bug gates passed on 2026-05-13:
  - branding CLI/TUI tests and live help checks;
  - LLMOps model catalog HTTP 200 with `object=list` and 17 models;
  - short `gena-debug exec` LLMOps smoke returned `OK`;
  - tool-loop smoke executed a shell command before final answer;
  - model/provider bootstrap tests;
  - Chat Completions API tests;
  - current `codex-core` Chat Completions tool-loop regression test;
  - TUI sidecar token rejection test.

## Blockers
- Manual TUI smoke is still pending because it requires interactive `/model` selection and one prompt.
- Release package/installer not rebuilt yet for `0.130.0`.

## Next Step
Run manual TUI smoke with `gena-debug`: open `/model`, confirm full LLMOps catalog, select a model, send `привет`, then build and verify release package/installer.
