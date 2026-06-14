# NOW

## Current Goal
Gena `0.139.0` installer is the only local installer artifact and `gena` resolves to `0.139.0`.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Code branch: `main`, clean against `origin/main`.
- Upstream `rust-v0.139.0` is merged into `main`; latest code commit is `d3fcb2e799 test: stabilize upstream 0.139 merge`.
- Local debug/release commands are `0.139.0`:
  - `gena-debug --version` -> `gena 0.139.0`;
  - `gena-release --version` -> `gena 0.139.0`.
- Installer artifact verified:
  - `codex-rs/dist/gena-v0.139.0-macos-arm64-installer.sh` (244M);
  - temp install produced `gena 0.139.0` and `gena-tui 0.139.0`;
  - checksum `b7d18563faa40841112ed61ef97b16bf2cf90606fcbf95319ffea66257e76925`.
- 2026-06-14 diagnosis:
  - stale `/Users/mntabunkov/.nvm/versions/node/v22.22.2/bin/gena` was `gena 0.130.0` and was earlier in `PATH` than user-local installs;
  - removed NVM stale `gena`/`gena.bin`;
  - removed old `codex-rs/dist/gena-v0.130.0-macos-arm64*` artifacts;
  - current `gena --version` resolves to `/opt/homebrew/bin/gena` -> `gena 0.139.0`.

## Blockers
- Real LLMOps smoke is blocked by transport/TLS to `https://devx-copilot.tech` from this machine:
  - catalog `curl` failed before HTTP with `LibreSSL SSL_connect: SSL_ERROR_SYSCALL`;
  - `gena-debug exec --oss --local-provider llmops ...` with token reached runtime but failed with transport disconnect.
- Manual interactive TUI prompt smoke still needs a real terminal.
- `codex-rs/dist` is ignored by git, so installer artifacts exist locally but are not committed.

## Next Step
When `devx-copilot.tech` is reachable again, rerun real LLMOps catalog plus `gena-release exec` smoke on installed `0.139.0`.
