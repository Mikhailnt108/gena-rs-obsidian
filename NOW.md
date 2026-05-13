# NOW

## Current Goal
Подготовить upstream update Gena до `rust-v0.130.0`.

## State
- Кодовый репозиторий: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Ветка кода: `chore/upstream-rust-v0.130.0`, запушена в `origin/chore/upstream-rust-v0.130.0`.
- Пользователь изменил delivery policy: PR не делать, после green gate мержить ветку напрямую в `main` и пушить `main`.
- Draft PR #1 существует из предыдущего шага, но дальше не используется как delivery path.
- Upstream tag: `rust-v0.130.0`.
- Merge commit: `3615ed38f4` (`Merge tag 'rust-v0.130.0'`).
- Сохранены Gena-specific изменения:
  - Gena `AGENTS.md`;
  - удаленные fork workflows;
  - CLI/TUI Gena branding;
  - LLMOps/runtime provider token onboarding;
  - Chat Completions compatibility adapter behavior.
- Диск был заполнен во время test link; очищено:
  - `codex-rs/target`;
  - старые `codex-rs/dist/gena-v0.125.0-macos-arm64*` artifacts.
- После полного test sweep диск снова заполнился; `codex-rs/target` удалён, сейчас около 73G свободно на `/System/Volumes/Data`.
- `git diff --check` по unstaged ручным правкам чистый.
- `git diff --cached --check` показывает trailing whitespace в upstream snapshot/patch content (`.snap`, `patches/v8_*`), не исправлялось, чтобы не менять snapshot/patch payload.
- Полный workspace test был запущен:
  - `CARGO_BUILD_JOBS=2 cargo test --workspace --all-targets --no-fail-fast`
  - включает все workspace crates из `codex-rs/Cargo.toml`, то есть и `codex-*`, и `gena-*`, плюс lib/bin/integration/doc-test targets.
- Gena-specific crates в полном прогоне прошли:
  - `gena-branding`, `gena-config`, `gena-plugin-api`, `gena-plugins-core`, `gena-runtime`, `gena-types`, `gena-upstream-adapter`.
- Исправлены deterministic upstream-merge test issues:
  - `codex-shell-command` redirect parser expectation restored;
  - `codex-network-proxy` duplicate contradictory WS/WSS proxy assertions removed;
  - `codex-mcp-server` test expected runtime package version instead of `0.0.0`;
  - MCP shell approval test made stable on macOS workspace-write sandbox.
- Дополнительно исправлены remaining `codex-core --test all` failures из прошлого состояния:
  - `suite::exec::openpty_works_under_real_exec_seatbelt_path` теперь допускает macOS `DARWIN_USER_TEMP_DIR` warning от system Python;
  - `suite::prompt_caching::*` ожидает test-shell `sh`, потому что `TestCodexBuilder` без `ShellZshFork` задаёт `/bin/sh`;
  - `suite::request_permissions_tool::approved_folder_write_request_permissions_unblocks_later_apply_patch` больше не ждёт лишний guardian retry response.
- Rechecked targeted tests:
  - `cargo test -p codex-shell-command bash::tests::parse_shell_lc_single_command_prefix_rejects_non_heredoc_redirects --lib` PASS;
  - `cargo test -p codex-network-proxy proxy::tests::apply_proxy_env_overrides_uses_plain_http_proxy_url --lib` PASS;
  - `cargo test -p codex-mcp-server --test all` PASS;
  - app-server tests that failed in full run pass individually.
- New targeted checks on 2026-05-12:
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::exec::openpty_works_under_real_exec_seatbelt_path -- --nocapture` PASS;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::prompt_caching -- --nocapture` PASS;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::request_permissions_tool::approved_folder_write_request_permissions_unblocks_later_apply_patch -- --nocapture` PASS;
  - `just fmt` PASS;
  - `just fix -p codex-core` PASS, with pre-existing `expect_used` warning in `core/tests/suite/shell_command.rs`.
- Stack overflow failures in `codex-core --lib` and `codex-tui --lib` pass when rerun with `RUST_MIN_STACK=16777216`.
- Full workspace rerun from 2026-05-12 was interrupted by user request:
  - `RUST_MIN_STACK=16777216 CARGO_BUILD_JOBS=2 cargo test --workspace --all-targets --no-fail-fast`
  - user requested interruption before completion; process was stopped with SIGINT during later `codex-tui` phase.
  - Failures observed before interruption:
    - `codex-app-server --lib`: `message_processor::message_processor_tracing_tests::thread_start_jsonrpc_span_exports_server_span_and_parents_children`;
    - `codex-app-server --test all`: `suite::v2::realtime_conversation::webrtc_v2_tool_call_delegated_turn_can_execute_shell_tool`, `suite::v2::turn_start::command_execution_notifications_include_process_id`;
    - `codex-core --lib`: `tools::runtimes::shell::unix_escalation::tests::execve_permission_request_hook_short_circuits_prompt`;
    - `codex-core --test all`: `suite::abort_tasks::interrupt_tool_records_history_entries`, `suite::approvals::spawned_subagent_execpolicy_amendment_propagates_to_parent_session`;
    - `codex-tui --lib`: 16 snapshot failures from version text `v0.128.0` -> `v0.130.0` and one oversized goal slash command snapshot.
  - Full rerun did confirm the newly fixed `codex-core --test all` targets pass inside the workspace run:
    - `suite::exec::openpty_works_under_real_exec_seatbelt_path`;
    - `suite::prompt_caching::*`;
    - `suite::request_permissions_tool::approved_folder_write_request_permissions_unblocks_later_apply_patch`.
- 2026-05-13 isolated reruns for observed failures all passed:
  - `cargo test -p codex-app-server --lib message_processor::message_processor_tracing_tests::thread_start_jsonrpc_span_exports_server_span_and_parents_children -- --nocapture` PASS;
  - `cargo test -p codex-app-server --test all suite::v2::realtime_conversation::webrtc_v2_tool_call_delegated_turn_can_execute_shell_tool -- --nocapture` PASS;
  - `cargo test -p codex-app-server --test all suite::v2::turn_start::command_execution_notifications_include_process_id -- --nocapture` PASS;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --lib tools::runtimes::shell::unix_escalation::tests::execve_permission_request_hook_short_circuits_prompt -- --nocapture` PASS;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::abort_tasks::interrupt_tool_records_history_entries -- --nocapture` PASS;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all suite::approvals::spawned_subagent_execpolicy_amendment_propagates_to_parent_session -- --nocapture` PASS.
- `codex-tui --lib` snapshots were inspected and accepted:
  - 15 status/history snapshots update expected version text from `v0.128.0` to `v0.130.0`;
  - `/goal` oversized objective snapshot updates formatted number separators from `4,001`/`4,000` to `4 001`/`4 000`, matching `codex_protocol::num_format::format_with_separators`;
  - `RUST_MIN_STACK=16777216 cargo test -p codex-tui --lib --no-fail-fast` PASS;
  - `cargo insta pending-snapshots --manifest-path tui/Cargo.toml` reports no pending snapshots.
- 2026-05-13 full workspace rerun completed:
  - `RUST_MIN_STACK=16777216 CARGO_BUILD_JOBS=2 cargo test --workspace --all-targets --no-fail-fast`
  - result: FAILED, 3 targets failed:
    - `-p codex-app-server --test all`;
    - `-p codex-core --lib`;
    - `-p codex-core --test all`.
  - failing tests were the same load-sensitive app-server/core areas and all pass when rerun individually.
- Current code diff is only accepted TUI snapshot updates.

## Blockers
- Full workspace test is still not green.
- Observed failures currently look load/timing sensitive because every failed test passes in isolation.
- Need decide next validation path: investigate/increase deadlines for load-sensitive tests or rerun full workspace under reduced test parallelism before direct merge to `main`.

## Next Step
1. Commit/push accepted TUI snapshot updates and this Obsidian checkpoint.
2. Choose next gate strategy for load-sensitive full workspace failures:
   - rerun full workspace with reduced test parallelism; or
   - harden the affected app-server/core tests against full-workspace load.
3. Once full gate is green, merge `chore/upstream-rust-v0.130.0` directly into `main` and push `main`; do not open/use PR as delivery path.
