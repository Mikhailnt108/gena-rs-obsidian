# NOW

## Current Goal
Gena `0.130.0` post-merge release package/installer built and verified.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Code branch: `main`, synced with `origin/main`, head `2734ce9d5f`.
- Upstream `rust-v0.130.0` is merged and pushed to `main`.
- Full workspace gate passed on 2026-05-13:
  - `RUST_MIN_STACK=16777216 CARGO_BUILD_JOBS=2 cargo test --workspace --all-targets --no-fail-fast -- --test-threads=1`
- Debug build/install and automated Gena bug gates passed on 2026-05-13.
- Manual TUI smoke was confirmed by user before release packaging:
  - `gena-debug`;
  - `/model`;
  - LLMOps catalog visible;
  - model selection and prompt checked manually.
- Release package built from current `main`:
  - `codex-rs/dist/gena-v0.130.0-macos-arm64/`
  - `codex-rs/dist/gena-v0.130.0-macos-arm64.tar.gz`
  - `codex-rs/dist/gena-v0.130.0-macos-arm64.tar.gz.sha256`
  - `codex-rs/dist/gena-v0.130.0-macos-arm64-installer.sh`
- Release verification passed:
  - packaged `gena --version` -> `gena 0.130.0`;
  - packaged `gena-tui --version` -> `gena-tui 0.130.0`;
  - tarball sha256 matches `.sha256`;
  - self-extract installer installed to a temp dir and wrappers reported `gena 0.130.0` / `gena-tui 0.130.0`.
- Shell `gena` without absolute path still resolves first to `/Users/mntabunkov/.nvm/versions/node/v22.22.2/bin/gena`, which reports old `0.128.0`; use explicit `/opt/homebrew/bin/gena` or adjust PATH before release install checks.
- Code repo is clean; `codex-rs/dist` artifacts are ignored by git.

## Blockers
- No blocker for local `0.130.0` release package/installer artifacts.

## Next Step
Decide whether to install the verified `0.130.0` release globally and whether to delete/archive old draft PR #1 plus branch `chore/upstream-rust-v0.130.0`.
