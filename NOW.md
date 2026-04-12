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
- `cargo check -p codex-cli` теперь доходит до `codex-cli`; текущий хвост уже узкий и локализован в login/manifest-слое `cli`.
- В `exec` уже добавлены `gena-runtime` / `gena-upstream-adapter` / `gena-plugins-core` зависимости и снят runtime/otel blocker.
- В `cli/src/login.rs` уже перевязаны runtime login helper'ы на `gena_runtime`, но сам `codex-cli` crate ещё не дотащил нужные `gena-*` зависимости в `Cargo.toml`.
- После финального merge по-прежнему нужен старый operational rule:
  - перед push дропнуть `.github/workflows/*` отдельным follow-up commit
- Локально `just` не установлен; пока использовался `cargo fmt --all`, но перед финализацией нужно вернуть обязательный `just fmt`.

## Blockers
- Merge ещё незакоммичен.
- `cargo check -p codex-cli` сейчас падает на отсутствующих `gena-*` зависимостях в `codex-rs/cli/Cargo.toml`.
- `just` отсутствует локально, а по процессу перед closeout нужен `just fmt`.

## Next Step
- Добавить `gena-runtime` и связанные `gena-*` зависимости в `codex-rs/cli/Cargo.toml`, затем повторить `cargo check -p codex-cli`.
