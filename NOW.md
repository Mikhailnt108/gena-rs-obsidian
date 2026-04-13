# NOW

## Current Goal
Довести реальный merge `openai/codex` `rust-v0.120.0` на ветке `update-upstream` до merge commit и запушить oauth-safe версию без `.github/workflows/*`.

## State
- Работа идёт в `gena-rs-project` на ветке `update-upstream`.
- В рабочее дерево уже наложен merge `rust-v0.120.0`, но merge commit пока не создан.
- Уже добиты ключевые швы совместимости:
  - `codex-api`
  - `codex-config`
  - `codex-core`
  - `app-server`
  - `gena-upstream-adapter`
  - `gena-runtime`
  - `codex-tui`
  - `codex-exec`
- `codex-cli` уже собран и протестирован после reconciliation в `cli`:
  - `cargo check -p codex-cli` PASS
  - `cargo test -p codex-cli` PASS
- Обязательные локальные cleanup steps уже частично пройдены:
  - `just fmt` PASS
  - `just fix` несколько раз прогонялся глубоко по workspace и уже внёс auto-fix правки в:
    - `codex-core`
    - `gena-runtime`
    - `codex-tui`
    - `codex-exec`
    - `codex-cli`
- Последние реальные blockers в `just fix` были не merge-конфликтами, а test-only API drift:
  - `core/tests/common/test_codex.rs`
  - `core/src/client_common_tests.rs`
  - `core/src/thread_manager_tests.rs`
  - `exec/src/lib_tests.rs`
- Эти test-only хвосты уже частично закрыты:
  - `ThreadManager::new(...)` в test helpers/tests доведён до новой сигнатуры с `config.model_provider.clone()`
  - `parallel_tool_calls` в `core/src/client_common_tests.rs` переведён на `Some(true)`
  - в `exec/src/lib_tests.rs` добавлен missing import `codex_core::config::ConfigBuilder`
- В `exec` уже добавлены `gena-runtime` / `gena-upstream-adapter` / `gena-plugins-core` зависимости и снят runtime/otel blocker.
- В `cli` уже сделан следующий слой правок:
  - `cli/Cargo.toml` получил `gena-branding` / `gena-plugins-core` / `gena-runtime` / `gena-types`
  - `cli/Cargo.toml` также получил `gena-upstream-adapter`
  - `cli/src/login.rs` перевязан на runtime login helper'ы из `gena_runtime`
  - `cli/src/mcp_cmd.rs` снова дотащен до runtime MCP helper layer
  - `cli/src/main.rs` reconciled с доконфликтной gena-версией настолько, чтобы `codex-cli` снова компилировался и проходил тесты:
    - возвращены debug plugin subcommands
    - восстановлен `build_multitool_command()`
    - убраны дубли `finalize_resume_interactive` / `finalize_fork_interactive`
    - починен dispatch для `cloud` и sandbox subcommands
- После финального merge по-прежнему нужен старый operational rule:
  - перед push дропнуть `.github/workflows/*` отдельным follow-up commit
- `just` уже установлен локально:
  - `cargo install just` PASS

## Blockers
- Merge ещё незакоммичен.
- `just fmt` уже пройден, но `just fix` ещё не доведён до финального чистого статуса.
- Нужен merge closeout:
  - merge commit
  - follow-up commit с drop `.github/workflows/*`
  - push ветки

## Next Step
- Допрогнать `just fix`, снять следующий точный test/lint хвост если он ещё всплывёт, затем перейти к merge closeout commit sequence для `update-upstream`.
