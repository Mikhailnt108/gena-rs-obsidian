# Gena Upstream Debug Test Gate

Version: v1
Applies from: `gena 0.125.0`

Этот документ фиксирует обязательный алгоритм upstream/update работы для `gena-rs`: сначала debug-сборка и Gena-specific проверки, затем только release. Codex upstream tests остаются обязательными, но не считаются достаточными для Gena.

## Цель

Не пропускать regression bugs, найденные на `0.125.0`:

- `/model` показывает только текущую модель вместо полного LLMOps catalog.
- LLMOps `/models` возвращает OpenAI-compatible `data[]`, а тесты мокают не тот формат.
- После выбора модели через `/model` Chat Completions получает 400 `System message must be at the beginning`.
- Chat Completions stream отдаёт `OutputTextDelta` до `OutputItemAdded` и роняет debug/TUI.
- `CODEX_HOME` / sidecar token path не создан или не используется.
- Release installer переустанавливает старое состояние до полного smoke-check.

## Основное правило

Release build запрещён, пока не пройдены:

1. Codex mandatory checks для затронутых crates.
2. Gena-specific automated gate.
3. Debug install.
4. Real LLMOps smoke через debug binary.
5. Manual TUI smoke через `gena-debug`.

Только после этого можно собирать release artifacts и installer.

## Upstream Algorithm

### 1. Prepare

- Проверить code repo:
  - `git status --short --branch`
  - `git log --oneline -5`
  - `git remote -v`
- Проверить Obsidian `NOW.md`.
- Проверить свободное место:
  - `df -h / /System/Volumes/Data codex-rs/target`
- Если свободно меньше 5GB:
  - не начинать release build
  - сначала чистить старые heavy artifacts.

### 2. Merge Upstream

- Работать на отдельной ветке для upstream, если это не small hotfix.
- Merge/cherry-pick upstream.
- Разрешить compile/API drift.
- Не размазывать Gena-specific fixes по upstream-owned code без необходимости.
- После merge проверить:
  - Gena binary targets существуют.
  - `gena-runtime`, `gena-upstream-adapter`, `gena-config`, `gena-*` deps не выпали.
  - default Gena provider остаётся `llmops`.
  - wrappers / installer scripts не потеряли `GENA_CODEX_HOME` / `CODEX_HOME` поведение.

### 3. Codex Mandatory Checks

В `codex-rs`:

- `just fmt`
- targeted crate tests по изменённым crates.
- `just fix -p <crate>` для затронутых Rust crates перед финализацией.
- Полный `cargo test` или `just test` только после явного решения, потому что он дорогой по времени и диску.

Эти проверки обязательны, но они не заменяют Gena gate.

## Gena Automated Gate

Запускать после upstream merge и после каждого Gena hotfix, если затронуты model/provider/chat/bootstrap/TUI paths.

### Models Catalog Gate

Цель: проверить реальный LLMOps catalog shape и sidecar token path.

Обязательные тесты:

- `cargo test -p gena-runtime gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured`
- `cargo test -p codex-api parses_openai_compatible_models_response`
- `cargo test -p codex-api parses_models_response`

Что обязано проверяться:

- response shape `{"object":"list","data":[...]}`.
- `id` -> model slug.
- `modelName` -> display name.
- текущая configured model не вытесняет remote catalog.
- sidecar token `CODEX_HOME/provider_tokens/LLMOPS_TOKEN` реально используется.
- request содержит LLMOps token headers.

### Provider Bootstrap Gate

Цель: проверить Gena startup/provider path, а не только isolated parser.

Обязательные тесты:

- `cargo test -p gena-runtime startup_model_tests`
- `cargo test -p gena-runtime`

Что обязано проверяться:

- `ThreadManager` / startup path использует фактический Gena provider, а не fallback `openai`.
- `CODEX_HOME` создаётся, если отсутствует.
- sidecar token читается без env token.
- first-run token prompt path не ломается.
- model cache refresh не скрывает LLMOps models.

### Chat Completions Gate

Цель: проверить поведение после выбора LLMOps модели через `/model`.

Обязательные тесты:

- `cargo test -p codex-core chat_completions_request_merges_instruction_messages_into_first_system_message`
- `cargo test -p codex-core chat_completion_text_stream_adds_item_before_text_delta`
- `cargo test -p codex-api endpoint::chat_completions::tests`

Что обязано проверяться:

- все `system` / `developer` instructions объединены в первый `system` message.
- после model switch нет отдельного developer/system message в середине Chat Completions payload.
- stream event order:
  - `Created`
  - `OutputItemAdded`
  - `OutputTextDelta`
  - `OutputItemDone`
  - `Completed`
- TUI не может получить `OutputTextDelta without active item`.

### Gena TUI Gate

Обязательные тесты:

- `cargo test -p codex-tui store_prompted_provider_token --lib`
- Если менялась `/model` UI логика:
  - `cargo test -p codex-tui model`
  - relevant `insta` snapshots для user-visible UI changes.

Что обязано проверяться:

- missing token prompt.
- empty token rejection.
- token persistence в sidecar.
- `/model` picker видит full catalog и current model одновременно.
- выбор модели не ломает следующий turn.

## Debug Build Gate

После automated gate:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project/codex-rs
cargo build -p codex-cli --bin gena -j 4
```

Установить debug явно:

```bash
install -m 0755 target/debug/gena /opt/homebrew/bin/gena-debug.bin
install -m 0755 target/debug/gena /opt/homebrew/bin/gena.bin
```

Wrapper `/opt/homebrew/bin/gena-debug` должен:

- выставлять `CODEX_HOME="${GENA_CODEX_HOME:-${HOME}/.gena-codex}"`, если он пустой.
- делать `mkdir -p "$CODEX_HOME"`.
- печатать, что запускается debug binary.
- exec в `/opt/homebrew/bin/gena-debug.bin`.

Проверить:

```bash
gena-debug --version
stat -f '%Sm %N' /opt/homebrew/bin/gena-debug.bin /opt/homebrew/bin/gena.bin
```

## Real LLMOps Smoke

Этот smoke обязателен, потому что mock tests уже пропускали реальные расхождения.

### Catalog Smoke

Без печати токена:

```bash
token="$(tr -d '\n\r' < ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)"
curl -sS -D /tmp/gena-llmops-headers.txt -o /tmp/gena-llmops-models.json \
  -H "Authorization: Bearer ${token}" \
  -H "X-Copilot-User-Token: ${token}" \
  -H "X-Copilot-User-Agent: devxcopilot/2.0" \
  https://devx-copilot.tech/v1/models
```

Проверить:

- HTTP 200.
- body содержит `object: "list"`.
- body содержит `data[]`.
- в `data[]` больше одной модели.

### Non-interactive Chat Smoke

Без печати токена:

```bash
LLMOPS_TOKEN="$(tr -d '\n\r' < ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)" \
gena-debug exec --oss --local-provider llmops \
  -m qwen3.5-35b-a3b \
  --sandbox read-only \
  --skip-git-repo-check \
  --color never \
  --json \
  --ephemeral \
  'Ответь ровно одним словом: OK'
```

Проверить:

- есть `item.completed`.
- ответ содержит `OK`.
- нет `System message must be at the beginning`.
- нет `OutputTextDelta without active item`.
- нет panic/backtrace.

## Manual TUI Smoke

Запустить:

```bash
gena-debug
```

Проверить header:

- `provider: llmops`.
- `model: gpt-oss-20b` или другой current model.
- нет warning про отсутствующий `CODEX_HOME`.

Дальше:

1. Выполнить `/model`.
2. Убедиться, что список содержит несколько LLMOps моделей, а не только current.
3. Выбрать LLMOps модель, например `qwen3.5-35b-a3b` или другую из списка.
4. Отправить:
   - `привет`
5. Проверить:
   - нет 400.
   - нет panic/backtrace.
   - нет зависания на бесконечном `Working`.
   - ответ отображается как обычное agent message.

Без этого release запрещён.

## Release Gate

Release build разрешён только если:

- Gena automated gate passed.
- Debug build installed.
- Real LLMOps smoke passed.
- Manual TUI smoke passed.
- Disk space enough for release artifacts.

После release build:

- пересобрать macOS arm64 tarball / sha256 / installer.
- установить release installer только после debug green path.
- повторить минимальный release smoke:
  - `gena --version`
  - first-run `CODEX_HOME` creation
  - `/model` full catalog
  - one prompt after model switch.

## Required Documentation Closeout

После upstream или Gena hotfix:

- `WORKLOG.md`:
  - commits
  - tests
  - debug install timestamp
  - real smoke result
- `NOW.md`:
  - current state
  - blockers
  - exactly one next action if following strict PROCESS v1
- `DL.md`:
  - any new decision or changed gate.
- Code repo:
  - commit and push.
- Obsidian repo:
  - commit and push.

## Failure Handling

Если баг найден на debug:

- Не собирать release.
- Добавить regression test, который воспроизводит реальный failure shape.
- Исправить код.
- Прогнать Gena automated gate заново.
- Пересобрать и переустановить debug.
- Повторить real smoke.

Если тест не совпадает с реальностью:

- Исправить тест первым.
- Mock должен отражать реальный wire payload или реальный event order.
- Не считать старый зелёный тест валидным доказательством.
