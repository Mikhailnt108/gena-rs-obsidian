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
- Нужен обязательный `just fmt` перед финализацией.
- Нужен merge closeout:
  - merge commit
  - follow-up commit с drop `.github/workflows/*`
  - push ветки

## Next Step
- Запустить `just fmt`, затем сделать merge closeout commit sequence для `update-upstream`.
