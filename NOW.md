# NOW

## Current Goal
Подготовить upstream update Gena до `rust-v0.130.0`.

## State
- Кодовый репозиторий: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Ветка кода: `chore/upstream-rust-v0.130.0`, запушена в `origin/chore/upstream-rust-v0.130.0`.
- Draft PR: https://github.com/Mikhailnt108/gena-rs-project/pull/1
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
- После полного прогона исправлены deterministic upstream-merge test issues:
  - `codex-shell-command` redirect parser expectation restored;
  - `codex-network-proxy` duplicate contradictory WS/WSS proxy assertions removed;
  - `codex-mcp-server` test expected runtime package version instead of `0.0.0`;
  - MCP shell approval test made stable on macOS workspace-write sandbox.
- Rechecked targeted tests:
  - `cargo test -p codex-shell-command bash::tests::parse_shell_lc_single_command_prefix_rejects_non_heredoc_redirects --lib` PASS;
  - `cargo test -p codex-network-proxy proxy::tests::apply_proxy_env_overrides_uses_plain_http_proxy_url --lib` PASS;
  - `cargo test -p codex-mcp-server --test all` PASS;
  - app-server tests that failed in full run pass individually.
- Stack overflow failures in `codex-core --lib` and `codex-tui --lib` pass when rerun with `RUST_MIN_STACK=16777216`.
- `RUST_MIN_STACK=16777216 cargo test -p codex-core --test all --no-fail-fast` no longer stack-overflows, but still has 6 failing tests:
  - `suite::exec::openpty_works_under_real_exec_seatbelt_path`;
  - `suite::prompt_caching::per_turn_overrides_keep_cached_prefix_and_key_constant`;
  - `suite::prompt_caching::prefixes_context_and_instructions_once_and_consistently_across_requests`;
  - `suite::prompt_caching::send_user_turn_with_changes_sends_environment_context`;
  - `suite::prompt_caching::send_user_turn_with_no_changes_does_not_send_environment_context`;
  - `suite::request_permissions_tool::approved_folder_write_request_permissions_unblocks_later_apply_patch`.

## Blockers
- Full workspace test still not green after upstream update; remaining stable failures are in `codex-core --test all`.

## Next Step
Commit/push the targeted test fixes, then investigate remaining `codex-core --test all` failures before marking PR #1 ready.
