# NOW

## Current Goal
Довести post-fix валидацию `gena + llmops` до продуктового финала: после закрытия startup disappearance и first-run onboarding остаётся только прямой UI smoke-check модели в picker.

## State
- В `gena-rs-project` активна ветка `main`.
- В `main` уже лежат оба связанных фикса для `gena + llmops`.
- Дополнительный фикс для `/model` вынесен в отдельную ветку:
  - `codex/llmops-model-header`
  - commit `bfe45e829` `fix(gena): send llmops token header for model refresh`
  - push выполнен в `origin/codex/llmops-model-header`
- Этот фикс добавляет для built-in `llmops` header:
  - `X-Copilot-User-Token` из `LLMOPS_TOKEN`
  - regression test, что Gena `/model` после refresh видит полный remote catalog, а не только `gpt-oss-20b (current)`
- `AGENT_BOOT.md` обновлён: macOS/Desktop путь к Obsidian теперь `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- В `main` зафиксирован commit:
  - `c714af8a5` `fix(gena): refresh llmops models with env auth`
- Дополнительно в `main` лежат:
  - `7a87e9437` `fix(gena): keep llmops models available on startup`
    - `ThreadManager::new()` теперь использует реально переданный provider
    - startup regression test добавлен в `gena-runtime`
  - `d5f73631d` `fix(gena): prompt for llmops token on first run`
    - при `RuntimeProviderTokenState::NeedsPrompt` TUI теперь спрашивает токен, сохраняет его в sidecar и продолжает старт
- Локально подтверждено:
  - `just fmt` PASS
  - `cargo test -p codex-models-manager` PASS
  - `cargo test -p gena-config` PASS
  - `cargo test -p gena-runtime` PASS
  - `cargo test -p codex-tui store_prompted_provider_token --lib` PASS
- Для `bfe45e829` подтверждено:
  - `just fmt` PASS
  - `cargo test -p codex-model-provider-info test_llmops_provider_sends_token_header_from_env` PASS в основном workspace
  - `cargo test -p gena-runtime startup_model_tests` PASS в основном workspace
  - повторная чистая сборка во временном worktree была остановлена нехваткой диска: `No space left on device`, при этом `git diff --check` PASS
- Release smoke-check на локально собранном `target/release/gena` подтверждает, что исходный баг с пропадающими `llmops`-моделями закрыт:
  - `provider: llmops` реально используется на старте
  - при наличии `LLMOPS_TOKEN` уходит `GET /v1/models?client_version=0.120.0`
  - запрос идёт с `Authorization: Bearer <token>`
  - в свежем `CODEX_HOME` создаётся `models_cache.json` с remote-моделью `env-provider-model`
- Отдельно подтверждён и first-run onboarding fix на release binary без `LLMOPS_TOKEN` в env:
  - появляется интерактивный prompt для `LLMOPS_TOKEN`
  - после ввода токена создаётся `provider_tokens/LLMOPS_TOKEN`
  - старт продолжается дальше до TUI, а затем создаётся `models_cache.json` с `client_version = 0.120.0`
- Дополнительно найден packaging drift:
  - старый `gena-v0.120.0-macos-arm64-installer.sh` из `Downloads` был собран до onboarding fix и переустанавливал pre-fix binary
- Актуальные macOS arm64 артефакты `gena-v0.120.0` пересобраны уже после onboarding fix:
  - `gena-v0.120.0-macos-arm64.tar.gz` обновлён в `2026-04-14 13:45:36`
  - `gena-v0.120.0-macos-arm64.tar.gz.sha256` обновлён в `2026-04-14 13:45:36`
  - `gena-v0.120.0-macos-arm64-installer.sh` пересоздан в `2026-04-14 13:45:43`
- Новый installer и новые release binaries локально установлены в `/opt/homebrew/bin`:
  - `gena 0.120.0`
  - `codex-tui 0.120.0`
- При запуске установленного `/opt/homebrew/bin/gena` на чистом `CODEX_HOME` без `LLMOPS_TOKEN` теперь подтверждается:
  - появляется prompt `Enter token for \`LLMOPS_TOKEN\``
- Исходный bug про startup disappearance `llmops`-моделей закрыт.
- First-run onboarding bug без `LLMOPS_TOKEN` тоже закрыт.
- Отдельный UI smoke-check через model picker всё ещё не закрыт.

## Blockers
- прямой UI smoke-check появления `llmops` моделей в picker пока не подтверждён

## Next Step
- Открыть PR из `codex/llmops-model-header` или перенести `bfe45e829` в нужную integration ветку.
- Затем прогнать прямой UI smoke-check в model picker и зафиксировать, что `llmops`-модели видны уже в интерактивном выборе моделей.
