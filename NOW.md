# NOW

## Current Goal
Validate fresh Gena `0.130.0` release built from current `main`.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Work is on `main`.
- Code repo is clean against `origin/main`; latest code commit is `ad1b9c822e`.
- Fresh release was rebuilt on 2026-05-19 from current `main`, including the Chat Completions contract fixes:
  - `e9ae97cab8`;
  - `874b8bca57`;
  - `9fd8534421`;
  - `ad1b9c822e`.
- Fresh local artifacts:
  - `codex-rs/dist/gena-v0.130.0-macos-arm64.tar.gz` mtime `2026-05-19 11:31:59 MSK`;
  - `codex-rs/dist/gena-v0.130.0-macos-arm64.tar.gz.sha256` mtime `2026-05-19 11:31:59 MSK`;
  - `codex-rs/dist/gena-v0.130.0-macos-arm64-installer.sh` mtime `2026-05-19 11:32:23 MSK`.
- Verification passed:
  - packaged `gena --version` -> `gena 0.130.0`;
  - packaged `gena-tui --version` -> `gena-tui 0.130.0`;
  - `shasum -a 256 -c` PASS;
  - installer `--help` PASS;
  - temp install smoke PASS.
- Fresh release installed to first PATH bin:
  - `/Users/mntabunkov/.nvm/versions/node/v22.22.2/bin`;
  - `gena --version` -> `gena 0.130.0`;
  - `gena-tui --version` -> `gena-tui 0.130.0`;
  - installed wrapper/bin mtimes are `2026-05-19 11:33:25 MSK`.
- Local cleanup on 2026-05-24:
  - deleted `codex-rs/target` build cache, freeing disk from 13 GiB to 69 GiB available;
  - deleted `/Users/mntabunkov/.cache/lm-studio`, freeing disk from 76 GiB to 89 GiB available;
  - preserved `codex-rs/dist/gena-v0.130.0-*` artifacts;
  - installed `gena 0.130.0` and `gena-tui 0.130.0` still run from PATH.

## Blockers
- Full workspace `cargo test` was not rerun for the release rebuild.
- Last known full workspace `cargo test` failed in unrelated `codex-app-server --test all` timeout tests:
  - `webrtc_v2_tool_call_delegated_turn_can_execute_shell_tool`;
  - `command_execution_notifications_include_process_id`.
- Real LLMOps validation may still be affected by intermittent TLS reset to `https://devx-copilot.tech`.

## Next Step
Run a real LLMOps smoke on installed `gena 0.130.0`.
