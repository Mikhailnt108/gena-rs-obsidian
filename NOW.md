# NOW

## Current Goal
Закрыть оставшийся first-run onboarding bug для `gena + llmops`: после подтверждения startup/model refresh fix осталось починить интерактивный запрос токена на первом запуске без `~/.gena-codex`.

## State
- В `gena-rs-project` активна ветка `main`.
- Рабочее дерево репозитория кода больше не чистое: локально лежат непушенные правки для дополнительного startup regression coverage.
- В `main` зафиксирован commit:
  - `c714af8a5` `fix(gena): refresh llmops models with env auth`
- После этого локально добит ещё один связанный fix, пока не закоммиченный:
  - `codex-rs/core/src/thread_manager.rs`
    - `ThreadManager::new()` теперь использует реально переданный provider, а не всегда `openai`
  - `codex-rs/gena-runtime/src/startup_model_tests.rs`
    - добавлен startup regression test для `gena + env-auth provider`
- Локально подтверждено:
  - `just fmt` PASS
  - `cargo test -p codex-models-manager` PASS
  - `cargo test -p gena-config` PASS
  - `cargo test -p gena-runtime` PASS
- Release smoke-check на локально собранном `target/release/gena` подтверждает, что исходный баг с пропадающими `llmops`-моделями закрыт:
  - `provider: llmops` реально используется на старте
  - при наличии `LLMOPS_TOKEN` уходит `GET /v1/models?client_version=0.120.0`
  - запрос идёт с `Authorization: Bearer <token>`
  - в свежем `CODEX_HOME` создаётся `models_cache.json` с remote-моделью `env-provider-model`
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
- Исходный баг про startup disappearance `llmops`-моделей больше не blocker.
- Актуальный blocker теперь другой runtime/onboarding bug:
  - если удалить `~/.gena-codex` и запустить `gena` впервые без `LLMOPS_TOKEN` в env,
    приложение не предлагает ввести токен
  - вместо этого TUI завершает старт сообщением:
    - `Provider \`llmops\` requires \`LLMOPS_TOKEN\`. Set it in the environment or persist a token before starting the TUI.`
- По коду причина локализована:
  - `gena-runtime::prepare_runtime_provider_token(...)` возвращает `NeedsPrompt`
  - `tui/src/lib.rs` не открывает prompt, а превращает этот кейс в fatal error
- Отдельный UI smoke-check через model picker всё ещё не закрыт.

## Blockers
- first-run onboarding для `llmops` сломан, если нет `LLMOPS_TOKEN` и нет sidecar token file
- пока этот bug не починен, новый пользовательский сценарий после удаления `~/.gena-codex` ломается ещё до нормального входа в TUI
- прямой UI smoke-check появления `llmops` моделей в picker пока не подтверждён

## Next Step
- Починить onboarding flow для `llmops`: на `RuntimeProviderTokenState::NeedsPrompt` TUI должен спрашивать токен и сохранять его, а не падать с ошибкой.
