# NOW

## Current Goal
Довести `gena + llmops` до подтверждённого пользовательского green path на debug-сборке, и только после этого пересобирать release artifacts.

## State
- Кодовый репозиторий: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Активная кодовая ветка: `main`.
- `main` синхронизирован с `origin/main`.
- Release после последних багфиксов не пересобирался намеренно: сейчас цикл проверки идёт через debug build.
- Для явного запуска debug создана команда:
  - `gena-debug`
  - wrapper: `/opt/homebrew/bin/gena-debug`
  - binary: `/opt/homebrew/bin/gena-debug.bin`
- Обычный `/opt/homebrew/bin/gena` сейчас тоже указывает на свежий debug binary, но для ручной проверки использовать именно `gena-debug`.

## Latest Code Commits
- `15bbd633b` `fix(gena): emit chat item before text delta`
  - Chat Completions adapter теперь отдаёт события в порядке `OutputItemAdded -> OutputTextDelta -> OutputItemDone`.
  - Закрывает debug panic `OutputTextDelta without active item` после ответа LLMOps.
- `2c9004753` `fix(gena): keep chat system message first`
  - Для Chat Completions все `system` / `developer` instructions сворачиваются в единственный первый `system` message.
  - Закрывает LLMOps 400: `System message must be at the beginning`.
- `2197cbef3` `fix(gena): parse llmops model catalog`
  - `/models` теперь парсит реальную LLMOps/OpenAI-compatible форму `{"object":"list","data":[...]}`.
  - Закрывает баг, где `/model` показывал только текущую `gpt-oss-20b`.
- `1cd2e69c3` `fix(gena): refresh llmops models with sidecar token`
  - Gena wrapper создаёт `CODEX_HOME`.
  - LLMOps token берётся из sidecar `~/.gena-codex/provider_tokens/LLMOPS_TOKEN`.
  - Для refresh добавлены нужные auth headers.

## Installed Debug
- Свежая debug-сборка установлена:
  - `/opt/homebrew/bin/gena-debug.bin` — `2026-05-02 15:06:34`
  - `/opt/homebrew/bin/gena.bin` — `2026-05-02 15:07:25`
- Проверка версии:
  - `gena-debug --version` -> `gena 0.125.0`

## Verified
- Реальный LLMOps `/v1/models` с sidecar token возвращает OpenAI-compatible `data[]` catalog.
- Реальный sanity-check через debug `gena exec` с моделью `qwen3.5-35b-a3b` прошёл:
  - ответ: `OK`
  - без `System message must be at the beginning`
  - без `OutputTextDelta without active item`
- Точечные тесты:
  - `cargo test -p gena-runtime gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured` PASS
  - `cargo test -p codex-api parses_openai_compatible_models_response` PASS
  - `cargo test -p codex-api parses_models_response` PASS
  - `cargo test -p codex-api endpoint::chat_completions::tests` PASS
  - `cargo test -p codex-core chat_completions_request_merges_instruction_messages_into_first_system_message` PASS
  - `cargo test -p codex-core chat_completion_text_stream_adds_item_before_text_delta` PASS
- Scoped fix/lint:
  - `just fix -p codex-api` PASS
  - `just fix -p codex-core` completed

## Important Operational Notes
- Не запускать `codex-rs/dist/gena-v0.125.0-macos-arm64-installer.sh` для текущей проверки: это release installer, а текущие фиксы валидируются через debug.
- Сборку release делать только после ручного green path в TUI:
  - `gena-debug`
  - `/model`
  - выбрать LLMOps модель из полного списка
  - отправить `привет`
  - убедиться, что нет 400, panic, зависания, и ответ отображается в TUI.
- На диске критически мало места:
  - `/System/Volumes/Data` показывает около `531MiB` свободно после последней debug установки.
  - Следующая сборка может снова упереться в `No space left on device`.

## Blockers
- Нужен ручной TUI smoke-check на свежем `gena-debug` после `15bbd633b`.
- Перед release желательно освободить место на диске.

## Next Step
- Запустить `gena-debug`.
- В TUI выбрать LLMOps модель через `/model`.
- Отправить короткий prompt.
- Если green path подтверждён, только тогда пересобирать release artifacts и installer.
