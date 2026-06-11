# NOW

## Current Goal
Gena `0.139.0` release installer is built and verified.

## State
- Code repo: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Work is on `main`.
- Upstream tag merged and pushed:
  - `rust-v0.139.0` (`a7dff90430`);
  - merge commit `c5edc3a4f1` — `Merge tag 'rust-v0.139.0'`;
  - `origin/main` advanced from `4fd1181d3e` to `c5edc3a4f1`.
- Local debug install refreshed on 2026-06-11:
  - `/Users/mntabunkov/.local/bin/gena-debug` -> `gena 0.139.0`;
  - `/Users/mntabunkov/.local/bin/gena-tui-debug` -> `gena-tui 0.139.0`;
  - copies also written to `/Users/mntabunkov/download/`.
- Local release install refreshed on 2026-06-11:
  - `/Users/mntabunkov/.local/bin/gena-release` -> `gena 0.139.0`;
  - `/Users/mntabunkov/.local/bin/gena-tui-release` -> `gena-tui 0.139.0`;
  - release binaries copied to `/Users/mntabunkov/download/gena-release` and `/Users/mntabunkov/download/gena-tui-release`;
  - build target: `/tmp/gena-target-release/release`.
- Installer artifact created on 2026-06-11:
  - `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project/codex-rs/dist/gena-v0.139.0-macos-arm64-installer.sh` (244M);
  - bundle `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project/codex-rs/dist/gena-v0.139.0-macos-arm64.tar.gz` (183M);
  - checksum `b7d18563faa40841112ed61ef97b16bf2cf90606fcbf95319ffea66257e76925`.
- Main compatibility resolutions:
  - kept Gena deletions for upstream workflow/setup files that were removed locally;
  - preserved Gena branding for `/exit`/`/quit` descriptions;
  - preserved Gena default LLMOps startup path in TUI;
  - adapted Gena runtime config/bootstrap to upstream `CloudConfigBundleLoader`;
  - adapted Gena runtime feature/config edits after upstream removed `ConfigEditsBuilder::with_profile`;
  - adapted memory clearing, thread fork, and turn `cwd` override to upstream APIs;
  - added new upstream `ModelInfo` fields in Gena model fallback paths;
  - kept Gena Chat Completions logging while preserving upstream responses-lite behavior.
- Verification passed for the merge:
  - `cargo metadata --format-version 1 --no-deps`;
  - `CARGO_BUILD_JOBS=4 cargo check -p codex-cli -p codex-tui -p codex-exec -p codex-models-manager -p gena-runtime -p gena-upstream-adapter`;
  - `just fmt`;
  - `just fix -p codex-api && just fix -p codex-models-manager && just fix -p codex-model-provider && just fix -p gena-runtime && just fix -p gena-upstream-adapter && just fix -p codex-tui && just fix -p codex-cli && just fix -p codex-core`;
  - `git diff --check`;
  - `cargo test -p codex-cli top_level_`;
  - `cargo test -p codex-tui top_level_`;
  - `cargo test -p codex-tui store_prompted_provider_token --lib`;
  - `cargo test -p gena-types detects_brands_from_program_stem`;
  - `cargo test -p gena-runtime startup_model_tests`;
  - `cargo test -p gena-runtime gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured`;
  - `cargo test -p gena-runtime`;
  - `cargo test -p codex-api parses_openai_compatible_models_response`;
  - `cargo test -p codex-api parses_models_response`;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::client::chat_completions_text_before_tool_call_runs_tool_loop_to_completion -- --nocapture`;
  - `cargo test -p codex-api endpoint::chat_completions::tests`;
  - `GENA_GLOBAL_BIN_DIR="$HOME/.local/bin" CARGO_BUILD_JOBS=4 codex-rs/scripts/build-and-install-gena.sh debug`;
  - `gena-debug --version`, `gena-debug --help`, `gena-debug plugin --help`;
  - `gena-tui-debug --version`, `gena-tui-debug --help`.
- Post-merge test stabilization was done after initial full-test failures:
  - added `aqa-codex` and `gena` doctor snapshots because `cargo test --workspace` runs all bin targets sharing `cli/src/main.rs`;
  - widened load-sensitive event/hook/test deadlines in app-server and core tests;
  - isolated git tests from global/system git config;
  - updated code-mode truncation expectations for upstream truncation policy behavior;
  - adjusted subagent/unified-exec tests to avoid exact request ordering and short event waits.
- After user asked to stop full reruns, the last failed tests were checked individually and passed:
  - `CARGO_BUILD_JOBS=4 RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::approvals::spawned_subagent_execpolicy_amendment_propagates_to_parent_session -- --nocapture`;
  - `CARGO_BUILD_JOBS=4 RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::subagent_notifications::encrypted_multi_agent_v2_spawn_sends_agent_message_to_child -- --nocapture`;
  - `CARGO_BUILD_JOBS=4 RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::unified_exec::unified_exec_emits_exec_command_end_event -- --nocapture`.
- Final local checks after these edits:
  - `just fmt` passed;
  - `just fix -p codex-core` passed;
  - `git diff --check` passed.
- `just fix -p codex-core` emitted one upstream test warning for `clippy::expect_used` in `core/tests/suite/shell_command.rs`, but returned exit code 0.
- Release build command passed:
  - `GENA_GLOBAL_BIN_DIR="$HOME/.local/bin" CARGO_BUILD_JOBS=4 codex-rs/scripts/build-and-install-gena.sh release`;
  - `gena-release --version`, `gena-tui-release --version`, and download-copy version checks passed;
  - `gena-release --help` and `gena-tui-release --help` passed.
- Installer verification passed:
  - `gena-v0.139.0-macos-arm64-installer.sh --help`;
  - temp install with `--install-dir <tmp> --no-path-hint`;
  - temp-installed `gena --version` -> `gena 0.139.0`;
  - temp-installed `gena-tui --version` -> `gena-tui 0.139.0`.

## Blockers
- Full workspace/core reruns were intentionally stopped after user asked to finish with tests and check only failed tests individually.
- Real LLMOps smoke is blocked by transport/TLS to `https://devx-copilot.tech` from this machine:
  - catalog `curl` fails before HTTP with `LibreSSL SSL_connect: SSL_ERROR_SYSCALL`;
  - `gena-debug exec --oss --local-provider llmops ...` without env token fails with `Missing environment variable: LLMOPS_TOKEN`;
  - `LLMOPS_TOKEN="$(cat ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)" gena-debug exec --oss --local-provider llmops -m qwen3.5-35b-a3b ...` reaches runtime, retries 5/5, then fails with `stream disconnected before completion: error sending request for url (https://devx-copilot.tech/v1/chat/completions)`;
  - no panic/backtrace, no `System message must be at the beginning`, no `OutputTextDelta without active item` observed.
- Manual interactive TUI prompt smoke was not run in this non-interactive CLI session; help/version smoke passed for `gena-tui-debug`.
- Release build emitted an unused-import warning for `gena_branding::current_cli_command_name` in `cli/src/main.rs`; build still succeeded.
- `codex-rs/dist` is ignored by git, so the installer exists locally but was not committed to the code repository.

## Next Step
Commit/push this Obsidian update. When `devx-copilot.tech` is reachable again, rerun real LLMOps catalog + `gena-release exec` smoke on installed `0.139.0`, and run an interactive `gena-tui-release` prompt smoke in a real terminal.
