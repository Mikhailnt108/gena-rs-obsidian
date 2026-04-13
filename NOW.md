# NOW

## Current Goal
Зафиксировать текущее post-fix состояние `gena-v0.120.0`: `llmops` refresh fix уже в `main`, но выявлен новый onboarding bug для первого запуска без `~/.gena-codex`.

## State
- В `gena-rs-project` активна ветка `main`.
- Рабочее дерево репозитория кода чистое (`git status` без изменений).
- В `main` зафиксирован commit:
  - `c714af8a5` `fix(gena): refresh llmops models with env auth`
- Смысл фикса:
  - `gena`/`llmops` теперь может online-refresh'ить `/models`, когда токен приходит через `env_key` (`LLMOPS_TOKEN`)
  - regression test добавлен в `codex-rs/models-manager`
- Локально подтверждено:
  - `just fmt` PASS
  - `cargo test -p codex-models-manager` PASS
- Пересобраны macOS arm64 артефакты `gena-v0.120.0`:
  - unpacked bundle обновлён в `2026-04-13 23:21`
  - `gena-v0.120.0-macos-arm64.tar.gz` обновлён в `2026-04-13 23:21`
  - `gena-v0.120.0-macos-arm64.tar.gz.sha256` обновлён в `2026-04-13 23:21`
  - `gena-v0.120.0-macos-arm64-installer.sh` пересоздан в `2026-04-13 23:25`
- Новый installer локально установлен в `/opt/homebrew/bin`:
  - `gena 0.120.0`
  - `codex-tui 0.120.0`
- При запуске установленного `gena` в header подтверждается:
  - `provider: llmops`
- Но найден новый runtime/onboarding bug:
  - если удалить `~/.gena-codex` и запустить `gena` впервые без `LLMOPS_TOKEN` в env,
    приложение не предлагает ввести токен
  - вместо этого TUI завершает старт сообщением:
    - `Provider \`llmops\` requires \`LLMOPS_TOKEN\`. Set it in the environment or persist a token before starting the TUI.`
- По коду причина локализована:
  - `gena-runtime::prepare_runtime_provider_token(...)` возвращает `NeedsPrompt`
  - `tui/src/lib.rs` не открывает prompt, а превращает этот кейс в fatal error
- Дополнительно: прямая smoke-check в model picker ещё не подтверждена.
  - `provider: llmops` уже виден
  - но `~/.gena-codex/models_cache.json` в текущей проверке не обновился и остался с `client_version = 0.107.0`

## Blockers
- Главный blocker сместился:
  - first-run onboarding для `llmops` сломан, если нет `LLMOPS_TOKEN` и нет sidecar token file
- Пока этот bug не починен, новый пользовательский сценарий после удаления `~/.gena-codex` ломается ещё до нормального входа в TUI.
- Отдельно остаётся неподтверждённым прямой UI smoke-check появления `llmops` моделей в picker.

## Next Step
- Починить onboarding flow для `llmops`: на `RuntimeProviderTokenState::NeedsPrompt` TUI должен спрашивать токен и сохранять его, а не падать с ошибкой.
