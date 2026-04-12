# NOW

## Current Goal
Довести реальный merge `openai/codex` `rust-v0.120.0` на ветке `update-upstream` до зелёного `cargo check`, затем закоммитить merge и запушить oauth-safe версию без `.github/workflows/*`.

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
- `cargo check -p codex-cli` уже дошёл до глубокого `codex-cli` merge-хвоста.
- В `exec` уже добавлены `gena-runtime` / `gena-upstream-adapter` / `gena-plugins-core` зависимости и снят runtime/otel blocker.
- В `cli` уже сделан следующий слой правок:
  - `cli/Cargo.toml` получил `gena-branding` / `gena-plugins-core` / `gena-runtime` / `gena-types`
  - `cli/src/login.rs` перевязан на runtime login helper'ы из `gena_runtime`
  - `cli/src/mcp_cmd.rs` снова дотащен до runtime MCP helper layer
  - `cli/src/main.rs` частично reconciled с доконфликтной gena-версией:
    - возвращены debug plugin subcommands
    - восстановлен `build_multitool_command()`
    - убраны дубли `finalize_resume_interactive` / `finalize_fork_interactive`
    - починен dispatch для `cloud` и sandbox subcommands
- После этих последних CLI правок новый полный `cargo check -p codex-cli` ещё не зафиксирован: последний повторный прогон был прерван до финального результата.
- После финального merge по-прежнему нужен старый operational rule:
  - перед push дропнуть `.github/workflows/*` отдельным follow-up commit
- Локально `just` не установлен; пока использовался `cargo fmt --all`, но перед финализацией нужно вернуть обязательный `just fmt`.

## Blockers
- Merge ещё незакоммичен.
- Полный свежий `cargo check -p codex-cli` после последних правок в `cli/main.rs` и `cli/mcp_cmd.rs` ещё не снят до конца.
- В `codex-cli` всё ещё остаётся merge pressure в `main.rs` и `mcp_cmd.rs`; точный остаток надо дочитать следующим прогоном сборки.
- `just` отсутствует локально, а по процессу перед closeout нужен `just fmt`.

## Next Step
- Повторить `cargo check -p codex-cli` после последних правок в `cli/main.rs` и `cli/mcp_cmd.rs`, затем добить следующий точный остаток merge-хвоста.
