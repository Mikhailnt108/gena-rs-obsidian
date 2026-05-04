# DL

## 2026-05-02 — `gena + llmops` release нельзя пересобирать раньше debug green path на реальном TUI
**Решение:**
- После повторных регрессов `llmops` больше не пересобирать release сразу после unit-level фикса.
- Сначала собирать и устанавливать debug binary:
  - `gena-debug`
  - `/opt/homebrew/bin/gena-debug.bin`
- Считать фикс готовым к release только после реального TUI smoke:
  - открыть `gena-debug`
  - выбрать LLMOps модель через `/model`
  - отправить короткий prompt
  - подтвердить отсутствие 400, panic, зависания и увидеть ответ в TUI.

**Причина:**
- Предыдущие тесты проверяли не реальность, а слишком идеализированный контракт:
  - `/models` mock использовал `{"models":[...]}`, хотя LLMOps реально отдаёт `{"object":"list","data":[...]}`
  - после выбора модели тесты не ловили Chat Completions payload, где instruction message оказывался не в допустимой позиции
  - stream adapter test не проверял порядок `OutputItemAdded` перед `OutputTextDelta`
- Release build занимает слишком много времени и диска, поэтому быстрый debug loop дешевле и честнее.

**Подтверждение:**
- Добавлены/обновлены regression tests:
  - `gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured`
  - `parses_openai_compatible_models_response`
  - `chat_completions_request_merges_instruction_messages_into_first_system_message`
  - `chat_completion_text_stream_adds_item_before_text_delta`
- Реальный LLMOps sanity-check на `qwen3.5-35b-a3b` через `gena-debug exec` вернул `OK` без 400 и без debug panic.

**Альтернативы:**
- Сразу пересобирать release после каждого фикса.
- Полагаться только на unit tests без live LLMOps smoke.
- Проверять только `/model` catalog, не проверяя последующий prompt после выбора модели.

## 2026-04-14 — `gena + llmops` нельзя считать закрытым без отдельной first-run onboarding проверки на release binary
**Решение:**
- Считать post-fix состояние `gena + llmops` завершённым только если подтверждены оба пользовательских сценария:
  - startup refresh / models cache
  - first-run onboarding без заранее заданного `LLMOPS_TOKEN`
- Для второго сценария обязательна именно release-проверка:
  - fresh `CODEX_HOME`
  - `LLMOPS_TOKEN` отсутствует в env
  - появляется prompt
  - токен сохраняется в `provider_tokens/LLMOPS_TOKEN`
  - старт продолжается дальше до TUI

**Причина:**
- Startup refresh fix и onboarding fix лежат в разных слоях.
- Даже после зелёного startup regression и release `/models` smoke пользователь всё ещё мог упираться в старый fatal error на первом запуске.
- Этот разрыв реально проявился на практике, пока onboarding fix был только в рабочем дереве и ещё не был доставлен до установленного бинаря.

**Подтверждение:**
- В `codex-rs/tui/src/lib.rs` добавлен интерактивный prompt flow для `RuntimeProviderTokenState::NeedsPrompt`.
- `cargo test -p codex-tui store_prompted_provider_token --lib` зелёный.
- Release smoke подтвердил prompt, sidecar persistence и дальнейший startup.

**Альтернативы:**
- Считать достаточным только startup/models-cache smoke.
- Ограничиться unit tests без ручной release-проверки.

## 2026-04-14 — Исчезновение `llmops`-моделей на старте `gena` нужно считать закрытым только после проверки не только `models-manager`, но и startup path + release smoke
**Решение:**
- Считать bug закрытым только при выполнении всех трёх слоёв проверки:
  - unit/integration regression в `codex-models-manager`
  - startup regression в `gena-runtime`
  - release smoke на реальном `gena` binary с `llmops`
- Для startup path опираться на два связанных исправления:
  - online refresh моделей должен работать при auth через `env_key`
  - `ThreadManager::new()` должен использовать фактически переданный provider, а не всегда `openai`

**Причина:**
- Один только зелёный тест в `models-manager` не доказывает, что `gena` реально проходит правильный provider path на старте.
- Во время проверки был найден второй баг глубже по стеку:
  - `ThreadManager::new()` игнорировал provider
  - из-за этого startup flow мог не использовать `llmops` даже после исправления env-auth refresh
- Только после release smoke стало видно полное поведение:
  - `GET /v1/models?client_version=0.120.0`
  - bearer auth из `LLMOPS_TOKEN`
  - запись remote-модели в fresh `models_cache.json`

**Подтверждение:**
- Локально зелёные:
  - `cargo test -p codex-models-manager`
  - `cargo test -p gena-config`
  - `cargo test -p gena-runtime`
- Новый startup regression test добавлен в:
  - `codex-rs/gena-runtime/src/startup_model_tests.rs`
- Release smoke на `target/release/gena` подтвердил, что `llmops`-каталог реально подтягивается и кэшируется на старте.

**Альтернативы:**
- Считать достаточным только unit-level проверку `models-manager`.
- Проверять только UI/model picker без отдельного startup/runtime regression слоя.

## 2026-03-19 — Для upstream sync на `chore/update-upstream-on-gena-arch` временным operational rule должен быть oauth-safe push без `.github/workflows/*`
**Решение:**
- Пока push идёт через OAuth remote без `workflow` scope, не пытаться пушить изменения в:
  - `.github/workflows/*`
- После merge `upstream/main`:
  - сначала завершать merge локально
  - затем делать follow-up commit, который возвращает `.github/workflows/*` к состоянию `origin/<branch>`
  - потом пушить уже oauth-safe branch state
- Использовать для этого явный commit message:
  - `sync(upstream): drop workflow changes for oauth push`

**Причина:**
- GitHub отклоняет push workflow-файлов без `workflow` scope.
- Сам upstream merge при этом может быть уже полностью разрешён и compile-green.
- Практически дешевле временно дропать workflow diff, чем перестраивать auth/remote прямо посреди sync-сессии.

**Подтверждение:**
- На `codex-rs/chore/update-upstream-on-gena-arch` merge `upstream/main` был завершён локально:
  - `9bc9e5253` — `Merge upstream/main into chore/update-upstream-on-gena-arch`
- Первый push был отклонён GitHub из-за workflow permissions.
- После follow-up commit:
  - `ae5f2141f` — `sync(upstream): drop workflow changes for oauth push`
- Ветка была успешно запушена в `origin`.

**Альтернативы:**
- Использовать remote/token с `workflow` scope и пушить полный diff.
- Ручной cherry-pick без workflow-файлов.
- Откладывать upstream sync до смены auth.

## 2026-03-19 — После formal closeout следующей проверкой должен стать реальный upstream update на отдельной ветке, а не новый runtime split
**Решение:**
- Не продолжать автоматически новый architecture cleanup.
- Создать отдельную кодовую ветку:
  - `chore/update-upstream-on-gena-arch`
- Использовать её как testbed для реального merge:
  - `upstream/main -> chore/update-upstream-on-gena-arch`
- Считать сам merge первой практической валидацией многодневной runtime/platform normalization.

**Причина:**
- После `Plugin Platform Expansion` phase дальнейшие split’ы уже давали diminishing returns.
- Лучшая проверка ценности всей работы теперь не новый refactor, а реальный upstream update.
- Только настоящий merge показывает, локализован ли fork-specific pressure в новых Gena-owned boundaries.

**Подтверждение:**
- Ветка создана и запушена:
  - `chore/update-upstream-on-gena-arch`
- Baseline checkpoint:
  - `4c8e65155` — `refactor(gena): split runtime bootstrap onboarding module`
- Merge `upstream/main` уже начат.
- Divergence перед merge:
  - `upstream/main`: `105`
  - `chore/update-upstream-on-gena-arch`: `187`
- Текущие conflict hotspots:
  - `.github/workflows/rust-release.yml`
  - `codex-rs/app-server-protocol/schema/json/ClientRequest.json`
  - `codex-rs/cli/src/mcp_cmd.rs`
  - `codex-rs/exec/Cargo.toml`
  - `codex-rs/exec/src/lib.rs`
  - `codex-rs/tui/src/app.rs`
  - `codex-rs/tui/src/chatwidget.rs`
  - `codex-rs/tui/src/slash_command.rs`

**Альтернативы:**
- Продолжать architecture cleanup без проверки merge reality.
- Открыть новую platform phase раньше времени.
- Ребейзить/мерджить `upstream/main` прямо в `chore/update-arch` без отдельной test branch.

## 2026-03-19 — После `4c8e65155` текущая Plugin Platform Expansion phase должна быть закрыта формально, а не продолжена по инерции
**Решение:**
- Не открывать автоматически новый runtime/code split frontier.
- Считать текущую `Plugin Platform Expansion` phase завершённой как архитектурный checkpoint.
- Следующий шаг, если работа продолжается, оформлять уже как новую отдельную фазу.

**Причина:**
- Последний audit и финальный bootstrap split показали, что remaining code splits уже дают diminishing returns.
- Основные Gena-owned runtime/platform boundaries уже оформлены.
- Plugin platform уже достигла достаточного execution/lifecycle/registry/distribution checkpoint для честного phase closeout.

**Подтверждение:**
- Formal closeout зафиксирован в Obsidian после checkpoint:
  - `4c8e65155` — `refactor(gena): split runtime bootstrap onboarding module`

**Альтернативы:**
- Продолжать делать новые code splits ради симметрии.
- Оставить фазу без явного closeout и растянуть задачу ещё дальше.

## 2026-03-19 — После config session split последним сильным runtime seam оказался bootstrap onboarding boundary
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/bootstrap_onboarding.rs`
- Вынести туда:
  - trust/login/onboarding gating
  - provider token prepare/persist
  - TUI-only bootstrap onboarding policy
- `bootstrap.rs` оставить за:
  - bootstrap context prep
  - OTEL/log_db bootstrap
  - OSS provider planning
- После этого не открывать автоматически новый code split frontier.

**Причина:**
- Final audit показал, что это последний remaining seam с нормальной отдачей.
- Вынесенный блок был TUI-only и архитектурно отличался от shared bootstrap context / OSS / observability helpers.
- Дальше новые split’ы уже выглядят как diminishing returns.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `4c8e65155` — `refactor(gena): split runtime bootstrap onboarding module`

**Альтернативы:**
- Продолжать резать `bootstrap.rs` дальше.
- Возвращаться к `config.rs`.
- Сразу идти в formal closeout без последнего runtime split.

## 2026-03-19 — После capability-axis split следующим шагом должен быть config session boundary, а не новый feature frontier
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/config_session.rs`
- Вынести туда:
  - session target resolution
  - latest session lookup
  - thread id resolution
  - session cwd lookup
  - turn-context cwd parsing
- `config.rs` оставить за:
  - config load/build/rebuild
  - config edits/persistence
  - policy/restrictions helpers
- Не открывать на этом шаге:
  - новый plugin feature frontier
  - generic config abstraction
  - auth/bootstrap expansion

**Причина:**
- После последних runtime split’ов `config.rs` оставался крупнейшим mixed boundary с реальным смешением responsibilities.
- Session/resume helpers уже образовывали отдельный связный seam в верхней части модуля.
- Этот split даёт structural payoff без открытия нового platform scope.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c3c1cf1e9` — `refactor(gena): split runtime config session module`

**Альтернативы:**
- Сразу идти в новый feature frontier.
- Резать `bootstrap.rs`.
- Оставить session helpers внутри `config.rs`.

## 2026-03-19 — После registry/parser split следующим шагом должен быть capability-axis boundary, а не новый feature frontier
**Решение:**
- Выделить отдельные runtime-owned модули:
  - `gena-runtime/src/capability_command.rs`
  - `gena-runtime/src/capability_provider.rs`
  - `gena-runtime/src/capability_tool.rs`
  - `gena-runtime/src/capability_workflow.rs`
- Вынести туда:
  - per-axis plan enums
  - per-axis resolve/require helpers
  - registry-based planning helpers
- `capabilities.rs` оставить за:
  - thin facade
  - re-export layer
  - tests
- Не открывать на этом шаге:
  - новый plugin feature frontier
  - generic execution abstraction
  - deeper config split

**Причина:**
- После последних runtime split’ов именно `capabilities.rs` оставался самым крупным mixed boundary.
- Он уже естественно группировался по 4 capability axes, поэтому split был дешёвым и структурно полезным.
- Это даёт новый payoff без открытия нового platform scope.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `88a9a60be` — `refactor(gena): split runtime capability axis modules`

**Альтернативы:**
- Сразу идти в новый feature frontier.
- Резать `config.rs`.
- Оставить all capability axes inside one module.

## 2026-03-19 — После registry service/model split следующим шагом должен быть registry parser boundary, а не новый feature frontier
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/registry_parser.rs`
- Вынести туда:
  - static TOML parsing
  - `[index]` / `[service]` / `[response]` validation
  - `[[packages]]` parsing
  - package entry parsing
- `registry.rs` оставить за:
  - inspect/search/install facade
  - page normalization and iteration
  - high-level registry facade
- Не открывать на этом шаге:
  - новый registry feature frontier
  - generic paginator abstraction
  - новый plugin execution scope

**Причина:**
- После split’ов `registry_model` и `registry_service` именно parser/validation слой оставался главным смешанным seam внутри `registry.rs`.
- Это был самый дешёвый remaining structural boundary в registry axis.
- Parser split уменьшает drift risk для static registry contract без открытия нового platform scope.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `03e793737` — `refactor(gena): split runtime registry parser module`

**Альтернативы:**
- Сразу идти в новый feature frontier.
- Делать generic paginator layer.
- Оставить parser внутри общего `registry.rs`.

## 2026-03-14 — После service/static split в `registry.rs` следующим шагом должен быть `registry_model.rs`, а не новый registry feature frontier
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/registry_model.rs`
- Вынести туда:
  - registry-facing structs
  - search result/page/page-info types
  - resolved package types
  - search scoring and fuzzy ranking helpers
- `registry.rs` оставить за:
  - static TOML index parsing
  - parser layer
  - static/search install facade
  - high-level registry facade
- Не открывать на этом шаге:
  - richer registry UX
  - новый installer frontier
  - новый execution frontier

**Причина:**
- После split’ов `registry_service/package_archive` именно registry-facing model + ranking внутри `registry.rs` оставались главным смешанным seam.
- Parser/service/model responsibilities уже достаточно оформились как отдельные зоны.
- Следующий highest-ROI шаг был явный split model/search boundary, а не новый feature frontier.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `a2f0f4627` — `refactor(gena): split runtime registry model module`

**Альтернативы:**
- Сразу идти в richer registry UX.
- Сразу открывать новый package frontier.
- Оставить registry-facing model внутри общего `registry.rs`.

## 2026-03-14 — После роста archive/remote URL слоя внутри `package.rs` следующим шагом должен быть `package_archive.rs`, а не новый installer frontier
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/package_archive.rs`
- Вынести туда:
  - archive inspect/install/replace/uninstall
  - remote archive URL install/inspect/replace/uninstall
  - archive manifest parsing/validation
  - extracted bundle resolution
- `package.rs` оставить за:
  - local library install/replace/uninstall
  - local bundle install/replace/uninstall
  - bundle discovery and validation helpers
  - high-level package facade
- Не открывать на этом шаге:
  - новый registry frontier
  - новый execution frontier
  - richer installer UX

**Причина:**
- После split’ов `transport/registry/registry_service/loader/execution/capabilities` именно archive+remote URL logic внутри `package.rs` оставалась главным смешанным seam.
- Local package ops и archive installer ops уже стали разными ответственностями.
- Следующий highest-ROI шаг был явный split archive boundary, а не новый installer feature frontier.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `9b62600e0` — `refactor(gena): split runtime package archive module`

**Альтернативы:**
- Сразу идти в richer installer UX.
- Сразу открывать новый registry frontier.
- Оставить archive logic внутри общего `package.rs`.

## 2026-03-14 — После service-backed growth внутри `registry.rs` следующим шагом должен быть `registry_service.rs`, а не ещё один paginator/helper step
**Решение:**
- Выделить отдельный runtime-owned module:
  - `gena-runtime/src/registry_service.rs`
- Вынести туда:
  - service-backed search
  - auth-aware service fetch
  - service-backed package resolution
  - service-side paging/navigation request construction
- `registry.rs` оставить за:
  - static TOML index parsing
  - static search/ranking
  - registry-facing structs
  - high-level facade
- Не открывать на этом шаге:
  - новый installer frontier
  - generic paginator abstraction
  - новый execution frontier

**Причина:**
- После split’ов `transport/package/loader/execution/capabilities` именно service-backed logic внутри `registry.rs` оставалась главным смешанным seam.
- Static index parsing и service-backed fetch/resolution уже стали разными ответственностями.
- Следующий highest-ROI шаг был явный split service client boundary, а не ещё один helper на paginator axis.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `f6bde621e` — `refactor(gena): split runtime registry service module`

**Альтернативы:**
- Продолжать paginator normalization.
- Сразу идти в новый installer frontier.
- Оставить service-backed logic внутри общего `registry.rs`.

## 2026-03-14 — После `execution.rs` следующим шагом должен быть `capabilities.rs`, а не новый feature frontier
**Решение:**
- После `727667207` выделить отдельный runtime-owned module:
  - `gena-runtime/src/capabilities.rs`
- Вынести туда:
  - command/provider/tool/workflow plan enums
  - registry-based capability planning helpers
  - public per-axis `resolve/require_*` API
- `plugins.rs` свести к thin compatibility shell.
- Не открывать на этом шаге:
  - новый execution frontier
  - generic planner abstraction
  - новый entrypoint cleanup

**Причина:**
- После split’ов `registry/package/transport/loader/execution` именно `plugins.rs` оставался главным толстым runtime surface.
- Capability planning уже стало отдельной ответственностью от:
  - loader/discovery
  - execution/lifecycle
  - registry/package/transport
- Следующий highest-ROI шаг был именно модульный split, а не новый feature frontier.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `cbcf72943` — `refactor(gena): split runtime capabilities module`

**Альтернативы:**
- Открыть новый execution frontier.
- Продолжать `plugins.rs` cleanup без нового module boundary.
- Сразу ставить formal checkpoint.

## 2026-03-14 — После paginator normalization следующим шагом должен быть явный registry service client module split, а не ещё одна paginator abstraction
**Решение:**
- После `b037dbd67` не продолжать micro-thinning paginator axis.
- Вместо этого выделить явный runtime-owned registry/service client boundary:
  - отдельный `registry.rs`
  - search/resolve/inspect/fetch/normalize logic вынести туда
- `plugins.rs` оставить за:
  - dynamic loader
  - execution/lifecycle
  - installer/archive flows

**Причина:**
- После search paging, cursor, iteration и normalization ось paginator/service response уже почти выжата по ROI.
- Следующий полезный шаг — не ещё одна abstraction на этой же оси, а явный boundary split.
- Это снижает связность `plugins.rs` и локализует registry/service evolution в отдельном runtime module.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `59f6a8ac2` — `refactor(gena): split runtime registry client module`

**Альтернативы:**
- Продолжать generic paginator abstraction.
- Делать ещё один service response helper.
- Оставить registry/service client внутри общего `plugins.rs`.

## 2026-03-14 — После runtime registry client split следующим шагом должен быть явный package management module split, а не новый execution/frontier
**Решение:**
- После `59f6a8ac2` не открывать сразу:
  - новый execution frontier
  - generic paginator abstraction
  - formal checkpoint
- Сначала выделить явный runtime-owned package management boundary:
  - отдельный `package.rs`
  - local library ops
  - bundle ops
  - archive ops
  - remote archive URL ops
  - bundle/archive discovery and validation helpers
- `plugins.rs` оставить за:
  - dynamic loader
  - execution/lifecycle
  - discovered runtime plugin state

**Причина:**
- После `registry.rs` split общий `plugins.rs` всё ещё держал второй крупный platform surface: installer/archive/package management.
- Следующий лучший шаг — не ещё один feature frontier, а симметричный boundary split для package story.
- Это локализует install/archive/remote package evolution отдельно от execution/lifecycle evolution.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `31afb59bf` — `refactor(gena): split runtime package module`

**Альтернативы:**
- Сразу открывать новый execution frontier.
- Делать formal checkpoint без package split.
- Оставить package management внутри общего `plugins.rs`.

## 2026-03-14 — После runtime package split следующим шагом должен быть shared transport/client module split, а не новый feature frontier
**Решение:**
- После `31afb59bf` не открывать сразу:
  - новый execution frontier
  - generic paginator abstraction
  - formal checkpoint
- Сначала выделить явный runtime-owned transport boundary:
  - отдельный `transport.rs`
  - shared HTTP response helper
  - text/bytes download helpers
  - auth-env bearer handling
  - shared `http://` / `https://` validation
- `registry.rs` и `package.rs` должны использовать этот слой, а не держать почти одинаковые transport helper’ы локально.

**Причина:**
- После `registry.rs` и `package.rs` split’ов дублирование transport logic стало уже явным:
  - scheme checks
  - `reqwest::blocking::Client`
  - bearer auth from env
  - text/bytes download helpers
- Следующий лучший шаг — не новый feature frontier, а локализация shared network policy в одном runtime module.
- Это снижает drift risk между registry/service story и archive/remote package story.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `cef80019f` — `refactor(gena): split runtime transport module`

**Альтернативы:**
- Сразу открывать новый execution frontier.
- Делать formal checkpoint без transport split.
- Оставить duplicated HTTP helper’ы в `registry.rs` и `package.rs`.

## 2026-03-14 — После runtime transport split следующим шагом должен быть plugin loader/discovery module split, а не новый execution frontier
**Решение:**
- После `cef80019f` не открывать сразу:
  - новый execution frontier
  - formal checkpoint
  - generic paginator abstraction
- Сначала выделить явный runtime-owned loader boundary:
  - отдельный `loader.rs`
  - static built-in registry init
  - dynamic loader discovery
  - env/configured library and directory resolution
  - dynamic plugin registration
  - dynamic manifest / bundle validation
- `plugins.rs` оставить за:
  - runtime snapshot
  - lifecycle manager
  - execution planning
  - resolve/require APIs

**Причина:**
- После `registry.rs`, `package.rs` и `transport.rs` split’ов самым большим и смешанным remaining surface оставался `plugins.rs`.
- Внутри него ещё одновременно жили:
  - loader/discovery
  - manifest validation
  - snapshot/lifecycle
  - execution planning
- Следующий лучший шаг — отделить loader/discovery от execution/lifecycle, а не открывать новый feature frontier.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `84e3da007` — `refactor(gena): split runtime loader module`

**Альтернативы:**
- Сразу открывать новый execution frontier.
- Делать formal checkpoint без loader split.
- Оставить loader/discovery внутри общего `plugins.rs`.

## 2026-03-14 — После runtime loader split следующим шагом должен быть execution/lifecycle module split, а не новый feature frontier
**Решение:**
- После `84e3da007` не открывать сразу:
  - новый feature frontier
  - formal checkpoint
  - capability expansion
- Сначала выделить явный runtime-owned execution boundary:
  - отдельный `execution.rs`
  - runtime snapshot
  - lifecycle manager
  - prepared execution
  - lifecycle trace
  - execution target / execution plan façade
- `plugins.rs` оставить за:
  - capability-specific plan enums
  - per-axis resolve/require helpers
  - built-in command/provider/tool/workflow planning surface

**Причина:**
- После `loader.rs` split самым большим remaining mixed surface оставался `plugins.rs`.
- Внутри него ещё одновременно жили:
  - snapshot/lifecycle
  - prepared execution
  - lifecycle trace
  - capability planning
- Следующий лучший шаг — отделить execution/lifecycle façade от capability planning, а не открывать новый frontier.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `727667207` — `refactor(gena): split runtime execution module`

**Альтернативы:**
- Сразу открывать новый feature frontier.
- Делать formal checkpoint без execution split.
- Оставить execution/lifecycle внутри общего `plugins.rs`.

## 2026-03-14 — После richer paginator response metadata следующим шагом должен быть normalized page view, а не generic paginator framework
**Решение:**
- После `b5764bafa` не переходить сразу к:
  - generic paginator abstraction
  - response middleware layer
  - service client subsystem
- Сначала добавить один runtime-owned normalized view:
  - `RuntimePluginSearchPageInfo`
  - `inspect_runtime_plugin_search_page(...)`
- CLI должен использовать этот view для `plugin-search-page` / `plugin-search-next`.

**Причина:**
- Response contract уже обогатился, но entrypoint всё ещё вручную собирал operational view из raw fields.
- Следующий cheapest useful step — normalization helper, а не ещё один framework layer.
- Это локализует paginator semantics в runtime и снижает payoff для дальнейшего micro-thinning этой оси.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `b037dbd67` — `refactor(gena): normalize paginator response view`

**Альтернативы:**
- Сразу идти в generic paginator framework.
- Сразу проектировать service client subsystem.
- Оставить CLI разбирать raw response fields вручную.

## 2026-03-14 — После bounded full page iteration следующим шагом должен быть richer response metadata, а не generic paginator framework
**Решение:**
- После `c52db4bb3` не переходить сразу к:
  - generic paginator abstraction
  - response normalization layer
  - background prefetch policy
- Сначала расширить optional `[response]` contract ещё двумя cheap operational fields:
  - `returned`
  - `has_more`
- Existing loop/navigation paths не менять.

**Причина:**
- Iteration уже работала, но response contract всё ещё был бедным для operator/debug UX.
- Следующий cheapest useful step — richer response metadata без нового control flow.
- Это подготавливает будущий normalization boundary, не открывая его раньше времени.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `b5764bafa` — `feat(gena): enrich paginator response metadata`

**Альтернативы:**
- Сразу проектировать generic paginator framework.
- Сразу делать response normalization layer.
- Оставить current response contract без `returned/has_more`.

## 2026-03-14 — После runtime-owned next-request resolver следующим шагом должен быть bounded full page iteration, а не generic paginator framework
**Решение:**
- После `168d9eace` не переходить сразу к:
  - generic paginator abstraction
  - service crawler framework
  - background prefetch
- Сначала добавить один bounded runtime-owned loop:
  - `RuntimePluginSearchIteration`
  - `search_runtime_plugin_registry_index_all_pages(...)`
- Loop должен:
  - reuse existing navigation contract
  - detect repeated requests
  - stop on `max_pages`
- CLI должен получить:
  - `--all-pages`
  - `--max-pages`

**Причина:**
- Navigation contract уже был завершён, но iteration всё ещё оставалась на стороне оператора.
- Следующий cheapest useful step — bounded multi-page loop поверх current contract.
- Это завершает paginator frontier без преждевременного generic framework.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c52db4bb3` — `feat(gena): add full page iteration for registry search`

**Альтернативы:**
- Сразу проектировать generic paginator framework.
- Сразу делать background prefetch.
- Оставить multi-page iteration только как manual operator flow.

## 2026-03-14 — После cursor-aware search contract следующим шагом должен быть runtime-owned next-request resolver, а не full paginator сразу
**Решение:**
- После `7ad1759fa` не переходить сразу к:
  - automatic page iteration loop
  - generic paginator framework
  - multi-call service crawler
- Сначала добавить один runtime-owned contract:
  - `RuntimePluginSearchNextRequest`
  - `resolve_runtime_plugin_search_next_request(...)`
- CLI должен только печатать next request, а не автоматически ходить за следующими страницами.

**Причина:**
- `cursor_param` и `next_cursor` уже сделали search continuation expressible.
- Следующий cheapest useful step — явный next-request contract поверх `next_page/page_size/next_cursor`.
- Это завершает navigation seam, не открывая ещё full pagination orchestration.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `168d9eace` — `feat(gena): add next-page search navigation`

**Альтернативы:**
- Сразу идти в automatic next-page loop.
- Сразу проектировать generic paginator framework.
- Оставить navigation semantics только в CLI output parsing.

## 2026-03-14 — После paged response metadata следующим шагом должен быть cursor-aware search contract, а не full navigation loop сразу
**Решение:**
- После `18c00538f` расширить service-backed search contract минимально:
  - optional `[service].cursor_param`
  - optional `[response].next_cursor`
- Добавить один cursor-aware runtime helper:
  - `search_runtime_plugin_registry_index_with_navigation(...)`
- Existing page-aware helper сохранить как совместимый wrapper.
- CLI должен получить:
  - `--cursor`
- Не открывать на этом шаге:
  - automatic next-page loop
  - multi-page iterator
  - cursor-based package resolution
  - generic paginator framework

**Причина:**
- Response metadata уже дала page context, но не дала service-friendly continuation token.
- Следующий cheapest useful step — cursor contract поверх уже собранного paging stack, не перепрыгивая сразу к full navigation loop.
- Это готовит следующий frontier без ломки existing page-based operator UX.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `7ad1759fa` — `feat(gena): add cursor-aware registry search`

**Альтернативы:**
- Сразу идти в full next-page loop.
- Сразу проектировать generic paginator framework.
- Оставить search только с `page/next_page`.

## 2026-03-14 — После lightweight ranking следующим шагом должен быть minimal fuzzy fallback, а не richer service layer
**Решение:**
- Не переходить сразу к:
  - richer registry service contract
  - external fuzzy matching dependency
  - dedicated search backend
- Сначала добавить lightweight fuzzy fallback в runtime search:
  - subsequence match on `package_id`
  - subsequence match on `display_name`
  - lower score than exact/prefix/substring matches

**Причина:**
- После ranking search UX уже была полезной, но всё ещё плохо ловила abbreviated queries.
- Следующий cheapest useful step — lightweight fuzzy fallback без открытия новой инфраструктуры.
- Это усиливает discoverability и всё ещё сохраняет простой local runtime implementation.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `18823ebd0` — `feat(gena): add fuzzy plugin search`

**Альтернативы:**
- Сразу проектировать richer registry service.
- Сразу тянуть внешнюю fuzzy dependency.
- Оставить ranked search без fuzzy fallback.

## 2026-03-14 — После minimal search UX следующим шагом должно быть lightweight ranking, а не сразу fuzzy/service layer
**Решение:**
- Не переходить сразу к:
  - fuzzy search engine
  - external ranking dependency
  - richer registry service contract
- Сначала сделать lightweight ranking поверх текущего search:
  - exact `package_id`
  - exact `display_name`
  - prefix match
  - substring match
  - CLI output должен печатать `score`

**Причина:**
- После basic search UX результаты уже были полезны, но неупорядочены.
- Следующий cheapest useful step — дать deterministic ranking без открытия новой инфраструктуры.
- Это улучшает operator UX и оставляет пространство для future fuzzy/service layer позже.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `a36ee1d93` — `feat(gena): rank plugin index search results`

**Альтернативы:**
- Сразу делать fuzzy search.
- Сразу проектировать richer registry service.
- Оставить search как plain substring filter.

## 2026-03-14 — После registry-backed management следующим шагом должен быть minimal search UX, а не сразу richer registry service
**Решение:**
- Не переходить сразу к:
  - ranking service
  - fuzzy search engine
  - richer registry backend contract
- Сначала добавить один minimal search layer:
  - search по remote index
  - substring match по:
    - `package_id`
    - `display_name`
  - CLI path:
    - `debug search-plugin-index <INDEX_URL> <QUERY>`

**Причина:**
- После inspect + install + replace + uninstall registry story уже имела action surface, но ещё не имела базового discoverability UX.
- Следующий cheapest useful step — search, но без немедленного открытия ranking/service complexity.
- Это завершает minimal operator flow вокруг remote plugin index.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `7fd710139` — `feat(gena): add plugin index search ux`

**Альтернативы:**
- Сразу делать fuzzy/ranked search.
- Сразу проектировать richer registry service.
- Оставить registry layer без search UX.

## 2026-03-14 — После registry-backed install следующим шагом должна быть management symmetry, а не сразу search UX
**Решение:**
- Не переходить сразу к:
  - search/discovery UX
  - richer registry query surface
  - installer manager
- Сначала довести registry-backed action loop до симметрии:
  - replace by package id
  - uninstall by package id

**Причина:**
- После `install-plugin-from-index` registry layer уже умела влиять на поведение, но оставалась асимметричной.
- Следующий cheapest useful step — замкнуть management loop до того же уровня, что и local/remote archive management.
- Это даёт clean base для следующего фронта уже в discoverability/search UX.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `56d8fa220` — `feat(gena): add registry-backed plugin management`

**Альтернативы:**
- Сразу идти в search/discovery UX.
- Оставить registry-backed layer install-only.
- Сразу проектировать richer registry service.

## 2026-03-14 — После read-only remote index inspection следующим шагом должен быть registry-backed install, а не сразу search service
**Решение:**
- Не переходить сразу к:
  - search/discovery API
  - richer registry UX
  - full installer manager
- Сначала добавить один minimal registry-backed action:
  - выбрать package по `package_id` из remote index
  - взять его `distribution_url`
  - переиспользовать existing remote install flow

**Причина:**
- После `inspect-plugin-index` registry story уже имела discoverability, но ещё не влияла на поведение.
- Следующий cheapest useful step — дать ей один реальный execution path: install by package id.
- Это лучший bridge к richer search/update UX, чем immediate jump to registry service complexity.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `5ded103dc` — `feat(gena): add registry-backed plugin install`

**Альтернативы:**
- Сразу идти в search/discovery UX.
- Сразу делать registry-backed update/uninstall.
- Оставить index layer read-only.

## 2026-03-14 — После remote archive management следующим шагом должен быть read-only remote index inspection, а не сразу registry service
**Решение:**
- Не переходить сразу к:
  - full registry/index service
  - package search API
  - installer manager with query integration
- Сначала добавить один read-only remote index path:
  - `inspect_runtime_plugin_registry_index_url(...)`
  - CLI:
    - `debug inspect-plugin-index <INDEX_URL>`
- Current contract сделать минимальным:
  - TOML
  - `[[packages]]`
  - `package_id`
  - `display_name`
  - `package_version`
  - `distribution_url`

**Причина:**
- После remote install/update/uninstall loop следующий cheapest useful step — discoverability.
- Read-only index inspection даёт first registry story без открытия service/search complexity.
- Это правильный bridge к registry-backed install или search UX, не таща сразу package manager.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `ae7d6f7ef` — `feat(gena): add remote plugin index inspection`

**Альтернативы:**
- Сразу идти в registry-backed install flow.
- Сразу делать search/discovery API.
- Оставить remote story без index/discoverability layer.

## 2026-03-14 — После first remote install path следующим шагом должна быть remote archive management symmetry, а не сразу registry service
**Решение:**
- Не переходить сразу к:
  - remote registry/index service
  - package search/discovery UX
  - full remote installer manager
- Сначала замкнуть remote archive management loop:
  - replace/update by URL
  - uninstall by URL

**Причина:**
- После `debug install-plugin-url` remote story уже имела настоящий networked install path, но оставалась асимметричной.
- Следующий cheapest useful step — довести remote loop до той же формы, что и local archive loop.
- Это даёт clean checkpoint перед registry/discovery story.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `149d4093b` — `feat(gena): add remote plugin archive management`

**Альтернативы:**
- Сразу открывать remote registry/index story.
- Сразу проектировать package search/discovery UX.
- Оставить remote story только как install-only path.

## 2026-03-14 — После package distribution metadata следующим шагом должен быть один реальный remote fetch/install path, а не сразу registry service
**Решение:**
- Не переходить сразу к:
  - registry/index service
  - remote update/uninstall symmetry
  - full remote installer manager
- Сначала добавить один узкий networked path:
  - runtime-owned remote archive download
  - затем reuse existing archive inspect/install flow
  - CLI path:
    - `debug install-plugin-url <ARCHIVE_URL>`

**Причина:**
- После `distribution_url` metadata contract remote story всё ещё была только декларативной.
- Следующий cheapest useful step — один реальный fetch/install path поверх уже готового archive/package layer.
- Это даёт настоящий remote installer checkpoint без немедленного открытия registry/distribution framework.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `0cc563f32` — `feat(gena): add remote plugin archive install`

**Альтернативы:**
- Сразу делать registry/index service story.
- Сразу добавлять remote update/uninstall symmetry.
- Оставить remote story на уровне metadata only.

## 2026-03-14 — После package-aware archive action output следующим шагом должен быть package-level distribution metadata contract, а не сразу downloader
**Решение:**
- Не переходить сразу к:
  - remote fetch/install flow
  - package registry service
  - full remote distribution manager
- Сначала расширить `gena-package.toml` первым remote-facing field:
  - `distribution_url`
- Runtime должен:
  - читать это поле при archive inspection
  - валидировать supported remote schemes:
    - `http://`
    - `https://`
  - возвращать `distribution_url` в installer-facing report

**Причина:**
- После package-aware archive action output local installer workflow уже достаточно зрелый.
- Следующий cheapest useful step для remote story — explicit distribution contract, а не downloader.
- Это открывает remote distribution semantics без сетевого scope и без premature installer manager.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `88dde7711` — `feat(gena): add package distribution metadata`

**Альтернативы:**
- Сразу делать remote fetch/install flow.
- Сразу идти в registry service story.
- Оставить remote origin only implicit through repository URLs in dynamic manifests.

## 2026-03-14 — После archive inspection следующим ecosystem step должен быть package-aware archive action output, а не сразу remote distribution
**Решение:**
- Не переходить сразу к:
  - remote distribution
  - downloader/install manager
  - registry/publishing service story
- Сначала сделать archive action output package-aware для:
  - install
  - replace
  - uninstall
- Эти CLI paths должны сначала inspect-ить archive contract и затем печатать:
  - `package_id`
  - `package_version`
  - `bundle_dir`
  - resulting filesystem path

**Причина:**
- После `inspect` installer UX уже умел preview package/bundle contract, но install/update/remove paths всё ещё возвращали только filesystem path.
- Следующий полезный шаг — сделать local archive workflow operator-visible и metadata-aware на всех archive actions.
- Это богаче, чем filesystem-only output, но заметно дешевле и чище, чем immediate jump to remote distribution.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `fe82e6a97` — `feat(gena): enrich archive installer output`

**Альтернативы:**
- Сразу идти в remote distribution.
- Сразу проектировать installer manager.
- Оставить package metadata только в `inspect`, но не в action outputs.

## 2026-03-14 — После richer archive metadata следующим ecosystem step должен быть archive inspection, а не сразу remote distribution
**Решение:**
- Не переходить сразу к:
  - remote distribution
  - downloader/install manager
  - package registry story
- Сначала добавить installer-facing preview path:
  - `inspect_runtime_plugin_bundle_archive(...)`
  - CLI path `debug inspect-plugin-archive <ARCHIVE_PATH>`
- Inspect должен возвращать:
  - archive package metadata
  - resolved bundle metadata
  - уже после validation archive contract

**Причина:**
- После richer archive metadata следующий полезный installer UX шаг — дать preview path до install, а не сразу открывать distribution scope.
- Это делает package contract observable и пригодным для human/operator validation.
- Такой шаг лучше подготавливает будущий installer workflow, чем immediate jump to remote distribution.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `f9693fcac` — `feat(gena): add plugin archive inspection`

**Альтернативы:**
- Сразу идти в remote distribution.
- Сразу делать installer manager.
- Оставить package contract only implicit during install/replace/uninstall.

## 2026-03-14 — После first archive package manifest следующим ecosystem step должен быть richer archive metadata, а не сразу remote distribution
**Решение:**
- Не переходить сразу к:
  - remote distribution
  - package registry story
  - full installer manager
- Сначала расширить `gena-package.toml` от routing-only contract до richer package metadata:
  - `package_id`
  - `display_name`
  - `package_version`
  - `bundle_dir`
- Archive install/update/uninstall должны валидировать эти поля против bundle metadata.

**Причина:**
- После `bundle_dir` archive layer уже умел разрешать bundle root, но ещё не имел explicit package identity/version contract.
- Такой шаг делает archive manifest ближе к real packaging model, не открывая distribution complexity.
- Это лучший мост к remote distribution или richer installer UX, чем immediate jump from routing-only manifest.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `84b100596` — `feat(gena): add richer archive metadata`

**Альтернативы:**
- Сразу идти в remote distribution.
- Сразу идти в package registry story.
- Оставить `gena-package.toml` только с `bundle_dir`.

## 2026-03-14 — После local archive loop следующим ecosystem step должен быть archive package manifest, а не немедленный переход в remote distribution
**Решение:**
- Не открывать сразу:
  - remote distribution
  - package registry story
  - richer installer manager
- Сначала добавить package-level archive contract:
  - `gena-package.toml`
  - field:
    - `bundle_dir = "..."`
- Archive install/update/uninstall должны:
  - сначала пытаться разрешить bundle root через package manifest
  - сохранять backward-compatible fallback на old one-bundle archive layout

**Причина:**
- После local archive install/update/uninstall loop archive layer уже имел UX, но ещё не имел явного package contract.
- `gena-package.toml` даёт следующий уровень структуры без открытия remote or registry complexity.
- Это лучший мост к richer package metadata и installer UX, чем сразу переходить к distribution story.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `a5877a023` — `feat(gena): add archive package manifest`

**Альтернативы:**
- Сразу идти в remote distribution.
- Сразу делать package registry story.
- Оставить archive semantics только на implicit one-bundle fallback.

## 2026-03-14 — После archive install/update следующим ecosystem step должен быть symmetric archive uninstall, чтобы замкнуть local archive loop
**Решение:**
- Не переходить сразу к:
  - richer installer manager
  - remote distribution
  - package registry story
- Сначала добавить symmetric archive removal path:
  - `uninstall_runtime_plugin_bundle_archive(...)`
  - CLI path `debug uninstall-plugin-archive <ARCHIVE_PATH>`

**Причина:**
- После archive install/update правильнее сначала замкнуть local archive management loop.
- Такой шаг переиспользует existing bundle uninstall semantics и не требует нового installer framework.
- Это даёт clean checkpoint, после которого следующая фаза уже действительно richer installer/package conventions, а не ещё один archive helper.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `92400f4ec` — `feat(gena): add plugin archive uninstall flow`

**Альтернативы:**
- Сразу идти в richer installer manager.
- Сразу идти в remote distribution.
- Оставить archive uninstall как manual bundle delete after archive inspection.

## 2026-03-14 — После first archive install UX следующим ecosystem step должен быть archive replace/update flow, а не немедленный installer manager
**Решение:**
- Не открывать сразу:
  - richer installer manager
  - archive uninstall framework
  - remote distribution
- Сначала добавить symmetric local update path поверх ZIP bundles:
  - `replace_runtime_plugin_bundle(...)`
  - `replace_runtime_plugin_bundle_archive(...)`
  - CLI path `debug replace-plugin-archive <ARCHIVE_PATH>`

**Причина:**
- После archive install следующий cheapest useful step — сделать archive path пригодным для update, а не только initial install.
- Такой шаг переиспользует уже собранные bundle validation/install boundaries и не требует отдельного package manager.
- Это даёт first archive install/update surface до перехода к richer installer conventions.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `31ada4214` — `feat(gena): add plugin archive replace flow`

**Альтернативы:**
- Сразу идти в archive uninstall.
- Сразу идти в richer installer manager.
- Сразу идти в remote distribution.

## 2026-03-14 — После bundle metadata validation следующим ecosystem step должен быть first archive install UX, а не немедленный installer/update framework
**Решение:**
- Не открывать сразу:
  - archive uninstall/update framework
  - remote distribution
  - richer installer manager
- Сначала добавить один минимальный archive-based path:
  - `install_runtime_plugin_bundle_archive(...)`
  - CLI path `debug install-plugin-archive <ARCHIVE_PATH>`
- Scope архива:
  - ZIP only
  - extract to temporary directory
  - require exactly one valid bundle root with `gena-plugin.toml`
  - reuse existing bundle install path into `<product_home>/plugins`

**Причина:**
- После bundle metadata validation следующий разумный шаг — сделать bundle transportable через архив, не открывая сразу весь installer/update layer.
- Такой шаг даёт первый archive/install UX и переиспользует уже собранные bundle validation/install boundaries.
- Это лучший incremental move, чем сразу проектировать symmetric archive management или remote distribution.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `17349f350` — `feat(gena): add plugin bundle archive install`

**Альтернативы:**
- Сразу идти в archive uninstall/update flow.
- Сразу идти в richer installer manager.
- Сразу идти в remote distribution.

## 2026-03-14 — После bundle install UX следующим ecosystem step должен быть bundle uninstall UX, а не ещё один шаг в archive layer
## 2026-03-14 — После local bundle UX следующим ecosystem step должен быть bundle metadata validation, а не мгновенный переход в archive installer
**Решение:**
- Не открывать сразу:
  - archive installer
  - remote distribution
  - richer package manager
- Сначала расширить `gena-plugin.toml` от layout-only manifest до identity/version contract:
  - `plugin_id`
  - `display_name`
  - `plugin_version`
  - `library`
- Dynamic loader должен:
  - читать bundle metadata при discovery
  - валидировать её против exported dynamic plugin manifest
  - включать bundle metadata в runtime/CLI reporting

**Причина:**
- После local install/uninstall loop у bundle layer уже был UX, но ещё не было identity contract.
- Перед archive/install UX полезнее сначала зафиксировать согласованность между bundle metadata и library manifest.
- Такой шаг даёт stronger packaging convention без открытия archive complexity раньше времени.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `b38c73607` — `feat(gena): add bundle metadata validation`

**Альтернативы:**
- Сразу идти в archive installer.
- Сразу идти в remote distribution.
- Оставить bundle manifest только layout-only (`library = "..."`) без identity/version contract.

## 2026-03-14 — После bundle install UX следующим ecosystem step должен быть bundle uninstall UX, а не ещё один шаг в archive layer
**Решение:**
- Не переходить сразу к:
  - archive installer
  - remote distribution
  - richer package manager
- Сначала добавить symmetric local bundle uninstall:
  - `uninstall_runtime_plugin_bundle(...)`
  - CLI path `debug uninstall-plugin-bundle <BUNDLE_DIR_OR_NAME>`
- Uninstall должен поддерживать:
  - absolute bundle path
  - relative bundle dir name inside `<product_home>/plugins`
  - validation of `gena-plugin.toml`

**Причина:**
- После bundle install UX логично сначала замкнуть local bundle management loop.
- Это завершает bundle-level local operations до перехода в archive/installer complexity.
- Такой шаг полезнее, чем сразу открывать archive layer при незавершённой local bundle surface.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `1c13e067b` — `feat(gena): add plugin bundle uninstall flow`

**Альтернативы:**
- Сразу идти в archive installer.
- Сразу идти в remote distribution.
- Оставить bundle uninstall как manual `rm -rf` вне runtime-owned path.

## 2026-03-14 — После bundle-directory convention следующим ecosystem step должен быть bundle install UX, а не мгновенный переход в archive manager
**Решение:**
- Не открывать сразу:
  - archive installer
  - remote/download flow
  - richer package manager
- Сначала добавить local bundle install UX:
  - `install_runtime_plugin_bundle(...)`
  - CLI path `debug install-plugin-bundle <BUNDLE_DIR>`
- Bundle install должен:
  - валидировать `gena-plugin.toml`
  - требовать valid `library = "..."`
  - копировать bundle directory в canonical `<product_home>/plugins`

**Причина:**
- После bundle discovery логично сначала получить bundle install path, а не сразу уходить в archive/download complexity.
- Это делает package convention реально используемой, а не только discoverable.
- Такой шаг лучше связывает package layout с installer UX при минимальном scope.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `50f75e0f2` — `feat(gena): add plugin bundle install flow`

**Альтернативы:**
- Сразу идти в archive manager.
- Сразу проектировать remote/download UX.
- Оставить bundle layout только для manual copy without runtime-owned install path.

## 2026-03-14 — После complete local management loop следующим ecosystem step должен быть bundle-directory package convention, а не сразу archive manager
**Решение:**
- Не открывать сразу:
  - archive format
  - installer UX
  - package manager
- Сначала добавить first package layout convention:
  - plugin bundle directory
  - `gena-plugin.toml`
  - manifest field `library = "..."`
- Dynamic loader должен уметь находить такие bundles рядом с raw shared libraries.

**Причина:**
- После install/replace/uninstall следующая правильная стадия — packaging layout, а не ещё один file-op helper.
- Bundle-directory convention даёт следующий уровень структуры, не открывая пока archive/install framework.
- Это лучший мост от local management loop к будущему installer/package UX.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `40bd71a84` — `feat(gena): add plugin bundle directory convention`

**Альтернативы:**
- Сразу идти в archive format.
- Сразу строить installer UX.
- Оставить plugin packaging только в виде raw shared libraries.

## 2026-03-14 — После local install/uninstall следующим ecosystem step должен быть replace flow, а не прыжок сразу в package/archive model
**Решение:**
- Не переходить сразу к:
  - archive/package format
  - installer UX
  - updater framework
- Сначала добавить symmetric local replace flow:
  - `replace_runtime_plugin_library(...)`
  - CLI path `debug replace-plugin <LIBRARY_PATH>`
- Replace должен:
  - target existing installed plugin by file name in `<product_home>/plugins`
  - fail if target is not already installed

**Причина:**
- После install/uninstall правильный следующий шаг — замкнуть local file-management loop.
- Это завершает базовую local management модель до того, как мы уйдём в richer package/archive or installer story.
- Такой порядок лучше, чем перескакивать в packaging layer с незавершённым local management surface.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `baf13201a` — `feat(gena): add minimal plugin replace flow`

**Альтернативы:**
- Сразу идти в archive/package convention.
- Сразу открывать installer/update UX.
- Оставить replace как implicit reinstall without explicit runtime-owned path.

## 2026-03-14 — После minimal install flow следующим ecosystem step должен быть symmetric uninstall, а не сразу updater/package manager
**Решение:**
- Не открывать сразу:
  - update manager
  - plugin replacement pipeline
  - package manager
- Сначала добавить symmetric local uninstall flow:
  - `uninstall_runtime_plugin_library(...)`
  - CLI path `debug uninstall-plugin <LIBRARY_PATH_OR_NAME>`
- Uninstall должен поддерживать:
  - absolute installed path
  - relative file name inside `<product_home>/plugins`

**Причина:**
- После minimal install flow следующий правильный шаг — замкнуть local plugin management loop.
- Это даёт реальный install/uninstall surface без преждевременного update/package framework.
- Такой шаг полезнее, чем перескакивать в updater до появления базового remove operation.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `1e7d7ae44` — `feat(gena): add minimal plugin uninstall flow`

**Альтернативы:**
- Сразу идти в updater/replace flow.
- Сразу проектировать package manager.
- Оставить uninstall как manual rm outside runtime-owned path.

## 2026-03-14 — После canonical install location следующим ecosystem step должен быть minimal install flow, а не сразу полноценный installer
**Решение:**
- Не открывать сразу:
  - package manager
  - download/install pipeline
  - update/uninstall framework
- Сначала добавить minimal runtime-owned install flow:
  - `install_runtime_plugin_library(...)`
  - CLI path `debug install-plugin <LIBRARY_PATH>`
- Install flow должен использовать canonical destination:
  - `<product_home>/plugins`

**Причина:**
- После появления canonical install location следующий правильный шаг — дать первый реальный install path, а не только layout convention.
- Это связывает distribution model с runtime loader без преждевременного installer framework.
- Такой шаг полезнее, чем сразу проектировать package manager без базового local install operation.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `084412c94` — `feat(gena): add minimal plugin install flow`

**Альтернативы:**
- Сразу идти в full installer.
- Сразу добавлять uninstall/update pipeline.
- Оставить install story только на manual copy without runtime-owned helper.

## 2026-03-14 — После directory-based discovery следующим ecosystem step должен быть canonical default install location, а не сохранение env-only model
**Решение:**
- Не оставлять external plugin story только на:
  - `GENA_PLUGIN_LIBRARIES`
  - `GENA_PLUGIN_DIRS`
- Добавить default brand-aware install location:
  - `<product_home>/plugins`
- `GENA_PLUGIN_DIRS` оставить как extension/override layer.

**Причина:**
- После directory discovery следующий правильный шаг — перестать требовать env configuration как единственный install convention.
- Canonical install location делает external plugin story ближе к реальному packaging/install model.
- Это полезнее, чем сразу проектировать installer flow без стандартного destination layout.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c515ff206` — `feat(gena): add default plugin install directory`

**Альтернативы:**
- Оставить install story env-only.
- Сразу открывать installer/manager flow.
- Сразу вводить package archive model без canonical install destination.

## 2026-03-14 — После publishing metadata следующим ecosystem step должен быть directory-based discovery, а не сразу install pipeline
**Решение:**
- Не открывать сразу:
  - plugin installer
  - package download flow
  - full distribution pipeline
- Сначала добавить first distribution convention к dynamic loader:
  - `GENA_PLUGIN_DIRS`
  - non-recursive discovery shared libraries inside configured directories
  - platform-aware extension filtering

**Причина:**
- После manifest metadata следующий правильный шаг — сделать loader пригодным для directory-based plugin installation layout.
- Это уже реальная install/distribution convention, но ещё без преждевременного package manager/install flow.
- Такой шаг лучше, чем сразу проектировать весь install pipeline без базового directory discovery model.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `90d256e14` — `feat(gena): discover dynamic plugins from directories`

**Альтернативы:**
- Сразу открывать installer flow.
- Сразу проектировать package download/update pipeline.
- Оставить dynamic loading только на explicit library paths.

## 2026-03-14 — После runtime reporting следующим ecosystem step должен быть publishing-facing metadata contract, а не сразу packaging/distribution framework
**Решение:**
- Не открывать сразу:
  - external package format
  - install flow
  - publishing pipeline
- Сначала расширить `DynamicPluginManifest` publishing-facing metadata:
  - `plugin_version`
  - `repository_url`
  - `license`
- `gena-runtime` должен валидировать эти поля.
- `codex debug plugins` должен показывать их как часть discovered dynamic plugin report.

**Причина:**
- После ABI validation и runtime reporting следующий правильный шаг — сделать manifest пригодным не только для loading, но и для будущей publishing story.
- Это создаёт реалистичный metadata contract для external plugins без преждевременного packaging framework.
- Такой порядок лучше, чем сразу проектировать distribution/install flow без достаточного manifest surface.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `68d07491c` — `feat(gena): add dynamic plugin publishing metadata`

**Альтернативы:**
- Сразу идти в packaging format.
- Сразу проектировать install/publish pipeline.
- Оставить publishing story без runtime-visible metadata contract.

## 2026-03-14 — После compatibility manifest следующим ecosystem step должен быть runtime reporting path для discovered dynamic plugins
**Решение:**
- Не переходить сразу к publishing flow или richer package model.
- Сначала сделать runtime-owned reporting seam поверх уже существующего dynamic manifest:
  - `RuntimeDynamicPlugin`
  - `RuntimePluginSnapshot::dynamic_plugins()`
  - CLI report в `codex debug plugins`
- Reporting должен показывать:
  - `plugin_id`
  - `plugin_display_name`
  - `abi_version`
  - `plugin_api_version`
  - `library_path`

**Причина:**
- После валидации manifest следующий правильный шаг — сделать external plugin state наблюдаемым.
- Это даёт первый реальный operator/debug path для ecosystem story без преждевременного publishing framework.
- Такой шаг полезнее, чем сразу вводить более тяжёлую metadata/distribution модель вслепую.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `83d98350f` — `feat(gena): report discovered dynamic plugins`

**Альтернативы:**
- Сразу идти в publishing flow.
- Сразу расширять manifest до richer package model без runtime reporting.

## 2026-03-14 — После dynamic loader первым ecosystem step должен быть compatibility manifest, а не сразу publishing или richer external plugin model
**Решение:**
- Добавить минимальный compatibility contract для dynamic plugins:
  - `DynamicPluginManifest`
  - `DynamicPluginManifestFn`
  - `DYNAMIC_PLUGIN_MANIFEST_SYMBOL`
  - `CURRENT_DYNAMIC_PLUGIN_ABI_VERSION`
- `gena-runtime` должен читать manifest до вызова registrar symbol и валидировать ABI.
- `gena-plugins-core` должен экспортировать canonical manifest как reference implementation.
- На этом шаге сознательно не открывать:
  - publishing flow
  - compatibility matrix
  - richer external plugin packaging

**Причина:**
- После появления dynamic loader следующий правильный шаг — перестать грузить плагины вслепую.
- Versioned manifest даёт первый реальный ecosystem compatibility seam, не открывая ещё весь publishing/distribution фронт.
- Это минимальный и архитектурно правильный мост от internal dynamic loading к future external plugins.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `366dc55f2` — `feat(gena): add dynamic plugin compatibility manifest`

**Альтернативы:**
- Сразу идти в publishing story.
- Сразу открывать richer metadata package model.
- Оставить dynamic loader без compatibility boundary.

## 2026-03-13 — Первым реальным tool execution path должен стать `apply_patch` approval flow в TUI, а не искусственный debug-only consumer
**Решение:**
- Открыть `Tool Execution Phase` через `apply_patch`, а не через новый synthetic path.
- Добавить:
  - `ToolExecutionKind`
  - `RuntimeToolExecutionPlan`
  - `Tool` variant в unified execution planning
- Перевести на этот слой первый production path:
  - `ApplyPatchApprovalRequest` в `tui/chatwidget`
- Сознательно не вводить на этом шаге:
  - provider execution
  - lifecycle callbacks
  - dynamic loader

**Причина:**
- `apply_patch` уже существует как реальный user-facing tool path и даёт честный execution seam.
- Это лучше, чем строить tool execution на debug-only или искусственном consumer.
- Шаг продолжает execution stack последовательно: capability -> planning -> lifecycle -> prepared execution -> real tool path.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `63897b3e0` — `feat(gena): add apply patch tool execution`

**Альтернативы:**
- Начинать с provider execution.
- Открывать lifecycle callbacks раньше первого real tool path.
- Делать debug-only tool execution вместо production path.

## 2026-03-13 — После prepared execution следующим шагом должен быть lifecycle trace, а не немедленные callbacks или tool/provider execution
**Решение:**
- Не открывать сразу:
  - lifecycle callbacks
  - tool execution
  - provider execution
- Сначала добавить минимальный lifecycle trace поверх prepared execution:
  - `RuntimeLifecycleStage`
  - `RuntimePreparedExecutionTrace`
  - `prepare_execution_with_lifecycle(...)`
- Перевести на lifecycle-aware path:
  - `codex mcp`
  - `codex review`
- Зафиксировать пока только минимальные стадии:
  - `BeforePrepare`
  - `AfterPrepare`

**Причина:**
- После lifecycle manager и prepared execution логично сначала получить trace-bearing lifecycle pipeline.
- Это углубляет execution framework без преждевременного открытия callback model или новых execution axes.
- Шаг остаётся узким, проверяемым и архитектурно последовательным.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `12fda2981` — `feat(gena): add plugin lifecycle trace`

**Альтернативы:**
- Сразу открывать lifecycle callbacks.
- Сразу открывать tool execution.
- Сразу открывать provider execution.

## 2026-03-13 — После lifecycle manager следующим шагом должен быть prepared execution unit, а не немедленный переход к tool/provider execution
**Решение:**
- Не переходить сразу к `tool` или `provider` execution.
- Добавить следующий уровень runtime execution framework:
  - `RuntimePreparedExecution`
  - `prepare_execution_unit(...)`
- Prepared execution unit должен связывать:
  - execution plan
  - owning plugin id
  - owning plugin display name
- Перевести на него production paths:
  - `codex mcp`
  - `codex review`
- Сознательно не вводить на этом шаге:
  - tool/provider execution
  - lifecycle hooks
  - dynamic loading

**Причина:**
- После lifecycle manager следующим логичным шагом является ownership-aware execution unit, а не новая capability execution axis.
- Это углубляет execution framework без преждевременного открытия следующего тяжёлого фронта.
- Шаг остаётся compile-checkable и архитектурно связным.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `bde3fa2e7` — `feat(gena): add prepared plugin execution`

**Альтернативы:**
- Сразу открывать tool execution.
- Сразу открывать provider execution.
- Переходить к lifecycle hooks без ownership-aware execution unit.

## 2026-03-13 — После unified execution planning следующий шаг должен быть lifecycle manager, а не немедленный tool/provider execution
**Решение:**
- Не расширять execution phase сразу на `tool` и `provider`.
- Сначала добавить runtime-owned lifecycle слой поверх уже собранного execution stack:
  - `RuntimePluginLifecycle`
  - `discover(...)`
  - `registry()`
  - `prepare_execution(...)`
- Перевести на него production/debug paths:
  - `codex mcp`
  - `codex review`
  - `codex debug plugins`
- Сознательно не вводить на этом шаге:
  - tool/provider execution
  - dynamic loading
  - lifecycle hooks

**Причина:**
- После unified execution planning следующим архитектурно правильным шагом является общий lifecycle слой, а не новая capability execution axis.
- Это делает execution stack более целостным и уменьшает риск нарастить capability-specific execution без общего каркаса.
- Шаг остаётся достаточно узким и compile-checkable.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `463b2bf1a` — `feat(gena): add plugin lifecycle manager`

**Альтернативы:**
- Сразу открывать tool execution.
- Сразу открывать provider execution.
- Остановиться на unified execution planning без lifecycle слоя.

## 2026-03-13 — После `RuntimePluginSnapshot` следующий execution шаг должен быть unified execution planning, а не новые разрозненные helper'ы по capability axes
**Решение:**
- Не продолжать execution phase через отдельные `require_*` helper'ы для каждой capability.
- Добавить поверх `RuntimePluginSnapshot` общий execution слой:
  - `RuntimeExecutionTarget`
  - `RuntimeExecutionPlan`
  - unified `resolve/require_execution(...)`
- Перевести на него production paths:
  - `codex mcp`
  - `codex review`
- Сознательно не вводить на этом шаге:
  - tool/provider execution
  - dynamic loading
  - lifecycle manager

**Причина:**
- После snapshot уже нужен был не ещё один локальный seam, а первый общий lifecycle/dispatch facade.
- Unified planning делает execution phase архитектурно связной, не открывая сразу тяжёлый framework.
- Это лучше подготавливает выбор следующего крупного фронта, чем рост ad-hoc helper'ов.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `b7871bd33` — `feat(gena): unify plugin execution planning`

**Альтернативы:**
- Продолжать добавлять отдельные `require_*` helper'ы по capability.
- Сразу строить lifecycle manager.
- Остановиться на `RuntimePluginSnapshot` без unified execution layer.

## 2026-03-13 — После первого workflow execution шага нужен общий runtime snapshot, а не ещё один локальный discovered helper
**Решение:**
- Не добавлять следующий execution шаг как ещё один отдельный discovered helper в CLI.
- Ввести общий runtime façade:
  - `RuntimePluginSnapshot`
  - discovered registry snapshot
  - unified access для command/workflow execution
- Перевести на него production/debug callsite'ы:
  - `codex mcp`
  - `codex review`
  - `codex debug plugins`
- Сознательно не вводить на этом шаге:
  - lifecycle manager
  - dynamic loading
  - общий capability execution framework для tool/provider

**Причина:**
- После static discovery и первого workflow execution шага discovery wiring начало расползаться по CLI callsite'ам.
- Snapshot даёт первый общий runtime-owned execution façade без premature framework complexity.
- Это лучше подготавливает следующий lifecycle/dispatch frontier, чем ещё один узкий execution step по отдельной axis.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `58a9a9597` — `feat(gena): add runtime plugin snapshot`

**Альтернативы:**
- Добавить ещё один локальный discovered helper для новой capability.
- Сразу строить lifecycle manager.
- Остановиться на разрозненных discovered helper'ах без общего snapshot.

## 2026-03-13 — Первый execution шаг после static discovery должен идти через workflow execution plan, а не через общий dynamic framework
**Решение:**
- Не открывать сразу общий plugin execution/lifecycle framework.
- После static discovery добавить только минимальный execution step для уже существующей workflow axis:
  - `WorkflowExecutionKind`
  - `RuntimeWorkflowExecutionPlan`
  - discovered/runtime workflow execution helpers
  - production path `codex review`, идущий через runtime-owned workflow execution plan
- Сознательно не вводить на этом шаге:
  - общий dynamic plugin execution framework
  - lifecycle manager
  - tool/provider execution semantics

**Причина:**
- Это даёт второй execution-aware capability поверх уже собранного discovered registry без резкого открытия большого scope.
- `review` уже существует как понятный user-facing workflow path, поэтому execution seam получается реальным, а не искусственным.
- Шаг сохраняет compile-checkable темп и не тащит весь orchestration слой в plugins.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c30911689` — `feat(gena): add workflow execution plan`

**Альтернативы:**
- Сразу строить общий execution/lifecycle framework.
- Ограничиться static discovery без execution step.
- Открывать вместо workflow execution tool/provider execution frontier.

## 2026-03-13 — Первый loader-discovery шаг для plugin layer должен быть static discovery через registrars, а не сразу dynamic loader
**Решение:**
- Не строить сразу полный dynamic loader/lifecycle framework.
- Добавить минимальный static discovery layer:
  - `PluginRegistrar`
  - `inventory` collection
  - runtime helper для discovered registry
  - discovered command dispatch для production path `mcp`
- Сознательно не вводить на этом шаге:
  - dynamic loading
  - lifecycle manager
  - полный capability execution framework

**Причина:**
- Это честный переход от manual callback registry к loader-discovery phase без раздувания scope.
- Static discovery остаётся compile-checkable и заметно дешевле, чем полный execution framework.
- Production path получает первый реальный discovery bridge, не ломая текущую CLI/runtime модель.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `fb42632d6` — `feat(gena): add static plugin discovery`
  - `5c0605928` — `build(gena): sync lockfile for static discovery`

**Альтернативы:**
- Сразу открывать full dynamic loader.
- Оставаться на callback-only registry.
- Возвращаться к новому runtime frontier вместо первого loader-discovery шага.

## 2026-03-13 — После `tool` axis следующей plugin capability category может быть `workflow`, но только в very-thin виде
**Решение:**
- После `tool` axis не переходить сразу к workflow execution model.
- Добавить только минимальный `workflow` capability layer:
  - `WorkflowDescriptor`
  - `RegisteredWorkflow`
  - built-in `ReviewWorkflowPlugin`
  - runtime registry/introspection seam
- Не вводить на этом шаге:
  - dynamic workflow execution
  - loader/discovery
  - runtime-owned workflow dispatch/execution framework

**Причина:**
- Это завершает базовый capability set plugin layer без форсирования тяжёлой plugin execution architecture.
- `review` — понятный user-facing workflow seam, который можно зарегистрировать как descriptor/introspection consumer.
- Шаг остаётся архитектурно полезным, но не тащит orchestration в plugin layer слишком рано.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `9888f2c7c` — `feat(gena): add workflow plugin category`

**Альтернативы:**
- Остановиться на `command + provider + tool`.
- Сразу строить workflow execution framework.
- Возвращаться к новому runtime orchestration frontier вместо расширения plugin capability set.

## 2026-03-13 — После `provider` axis следующей plugin capability category может быть `tool`, но только в very-thin виде
**Решение:**
- После `provider` axis не переходить сразу к tool execution framework.
- Добавить только минимальный `tool` capability layer:
  - `ToolDescriptor`
  - `RegisteredTool`
  - built-in `ApplyPatchToolPlugin`
  - runtime registry/introspection seam
- Не вводить на этом шаге:
  - dynamic tool execution
  - loader/discovery
  - runtime-owned tool dispatch/execution framework

**Причина:**
- Это даёт следующий capability axis и двигает plugin architecture дальше `command + provider`.
- `apply_patch` — стабильный built-in tool seam, который можно зарегистрировать без перевода всей tool semantics в plugins.
- Шаг остаётся архитектурно полезным, но не открывает тяжёлый execution-фронт.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c1c260598` — `feat(gena): add tool plugin category`

**Альтернативы:**
- Остановиться на formal plugin checkpoint после `provider` axis.
- Сразу строить tool execution framework.
- Выбирать вместо `tool` новый runtime orchestration frontier.

## 2026-03-12 — После шага `de7e8ac69` plugin layer нужно считать formal checkpoint, а не продолжать инерционно ту же ось
**Решение:**
- Зафиксировать plugin layer как отдельный checkpoint после:
  - command-axis с runtime-owned dispatch plan
  - provider-axis как второй thin capability category
- Не форсировать сразу:
  - второй provider step ради симметрии
  - deeper provider integration
  - следующий plugin шаг без нового capability/frontier

**Причина:**
- Plugin layer уже вышел из состояния prework.
- Следующий маленький шаг по текущей оси даст мало новой архитектурной информации.
- Рациональнее открывать дальше уже новый capability/frontier как отдельный этап.

**Подтверждение:**
- Чекпоинт опирается на опубликованные шаги:
  - `441b095ef` — `feat(gena): add plugin command dispatch plan`
  - `de7e8ac69` — `feat(gena): add provider plugin category`

**Альтернативы:**
- Продолжать provider-axis ещё одним thin шагом ради симметрии.
- Открывать deeper provider integration без нового отдельного scope.

## 2026-03-12 — Первой новой plugin capability category после command-axis должна быть `provider`, но в very-thin виде
**Решение:**
- После закрытия migration-phase не форсировать второй command consumer.
- Открыть новый отдельный этап через `provider` category.
- Взять только минимальный scope:
  - `ProviderDescriptor`
  - `RegisteredProvider`
  - built-in `LlmOpsProviderPlugin`
  - runtime registry/introspection seam
- Не вводить на этом шаге:
  - dynamic provider execution
  - loader/discovery
  - plugin-owned bootstrap/model-selection/provider readiness logic

**Причина:**
- `provider` лучше продолжает уже существующие runtime/bootstrap/config/provider seams, чем `tool` или `workflow`.
- Шаг даёт новый capability axis и двигает нас к full target architecture из `GENA_REPO_STRUCTURE.md`.
- При этом он остаётся достаточно дешёвым и не открывает тяжёлый execution-фронт.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `de7e8ac69` — `feat(gena): add provider plugin category`

**Альтернативы:**
- Добавлять второй command consumer вместо новой category.
- Начинать сразу `tool` category.
- Форсировать dynamic provider execution слишком рано.

## 2026-03-12 — После `plugin command dispatch plan` текущую архитектурную migration-phase можно считать завершённой
**Решение:**
- Считать текущую migration-phase завершённой после шага `441b095ef`.
- Не форсировать внутри этой же задачи:
  - plugin loader/discovery,
  - dynamic execution,
  - второй built-in consumer,
  - новый runtime orchestration frontier.
- Formalization phase завершить фиксацией достигнутого DoD и границ scope в Obsidian.

**Причина:**
- Цель текущей задачи была не в полной plugin platform, а в переводе форка на Gena-owned архитектурные слои и заметном удешевлении upstream sync.
- Этот результат уже достигнут:
  - upstream-sensitive branding/config/bootstrap/auth/session/plugin seams локализованы намного лучше, чем в начале;
  - plugin phase дошла до первого реального рабочего состояния, а не осталась только prework-слоем.
- Дальнейшие шаги уже открывают новый scope, а не завершают текущий.

**Подтверждение:**
- Финальный кодовый checkpoint текущей фазы:
  - `441b095ef` — `feat(gena): add plugin command dispatch plan`

**Альтернативы:**
- Продолжать plugin phase до loader/discovery прямо сейчас.
- Добавлять ещё один consumer только ради полноты.
- Смешивать закрытие текущей migration-phase с новым runtime orchestration этапом.

## 2026-03-11 — После thin integration seam нужен первый реальный usage path для registry, но всё ещё не execution framework
**Решение:**
- Не переходить сразу к plugin execution path или loader.
- Добавить маленький CLI-facing usage path:
  - `codex debug plugins`
  - печать registered built-in plugins и command names из core registry
- Использовать для этого:
  - `gena-plugins-core::build_core_plugin_registry()`

**Причина:**
- Это первый реальный consumer registry за пределами тестов и внутренних helper'ов.
- Шаг остаётся маленьким и compile-checkable.
- Он подтверждает, что registry уже можно использовать в пользовательском коде, не открывая полную plugin system.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `491b491bd` — `feat(gena): expose plugin registry via debug command`

**Альтернативы:**
- Сразу идти в execution path.
- Ограничиться только internal integration seam без первого user-facing usage path.

## 2026-03-11 — После первого built-in consumer следующим шагом должен быть very-thin integration seam, но всё ещё без loader/discovery
**Решение:**
- После `gena-plugins-core` не переходить сразу к полноценной runtime plugin system.
- Добавить только очень тонкую интеграцию:
  - `gena-runtime::initialize_runtime_plugin_registry(...)`
  - `gena-plugins-core::build_core_plugin_registry()`
- Оставить за рамками этого шага:
  - plugin loader
  - discovery
  - реальный execution path

**Причина:**
- Это даёт первый настоящий мост между runtime-слоем и built-in plugin consumer.
- Шаг остаётся маленьким и compile-checkable.
- Он сохраняет чистое dependency direction: `gena-plugins-core -> gena-runtime public helper`, а не наоборот.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `95a341185` — `feat(gena): add thin plugin runtime integration`

**Альтернативы:**
- Сразу вводить runtime loader/discovery.
- Продолжать только добавлять новые plugin descriptors без первого integration seam.

## 2026-03-11 — После `PluginRegistry` следующим шагом в `plugin-api prework` должен быть первый реальный consumer, а не сразу runtime loader
**Решение:**
- Не переходить сразу к runtime plugin loader или discovery.
- Добавить первый built-in consumer в отдельный crate `gena-plugins-core`.
- Взять максимально узкий consumer:
  - `McpCommandPlugin`
  - `register_core_plugins(...)`
- Пока не интегрировать этот consumer в `cli` или `gena-runtime`.

**Причина:**
- Это первый реальный proof that the API is usable.
- Шаг остаётся маленьким и compile-checkable.
- Он продолжает двигать нас к target architecture, не открывая слишком широкий фронт изменений.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `43e301d50` — `feat(gena): add core command plugin consumer`

**Альтернативы:**
- Сразу запускать runtime loader / discovery integration.
- Добавлять второй и третий consumer без первого минимального proof step.

## 2026-03-11 — Следующий шаг после минимального `gena-plugin-api` crate: добавить registry seam, но всё ещё не форсировать runtime plugin loader
**Решение:**
- Не оставлять `gena-plugin-api` только набором trait names.
- Добавить минимальный registration contract:
  - `PluginRegistry`
  - default `register(...)` methods на plugin traits
- При этом не вводить:
  - runtime plugin loader
  - plugin discovery
  - integration с `gena-runtime`

**Причина:**
- Это делает `plugin-api` архитектурно осмысленным, а не purely declarative.
- Шаг остаётся маленьким и compile-checkable.
- Он подготавливает следующий шаг: выбор первого реального consumer seam.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `37fd7c481` — `feat(gena): add plugin registry seam`

**Альтернативы:**
- Оставить `gena-plugin-api` без registration contract.
- Сразу переходить к runtime loader или discovery mechanism.

## 2026-03-11 — После исчерпания cheap production seams следующий рациональный шаг: very-thin `plugin-api prework`, а не преждевременная полноценная plugin system
**Решение:**
- После закрытия `debug_sandbox` config seam не продолжать охоту за всё более мелкими local helper'ами.
- Вместо этого начать следующую фазу target architecture:
  - добавить отдельный crate `gena-plugin-api`
  - держать его очень маленьким
  - не интегрировать сразу plugin loader или runtime registry
- Зафиксировать пока только минимальные stable contracts:
  - `PluginMetadata`
  - `Plugin`
  - `ToolPlugin`
  - `WorkflowPlugin`
  - `ProviderPlugin`
  - `MemoryPlugin`
  - `CommandPlugin`

**Причина:**
- В production `cli/tui/exec` cheap shared seams почти закончились.
- Следующий полезный шаг уже должен двигать нас к `GENA_REPO_STRUCTURE`, а не только снижать merge-cost локальными cleanup'ами.
- Минимальный crate даёт архитектурную опору без premature runtime integration.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `381275e4a` — `feat(gena): add minimal plugin api crate`

**Альтернативы:**
- Продолжать выжимать всё более мелкие local seams из production entrypoints.
- Сразу запускать полноценную plugin system с loader/registry и широкими runtime changes.

## 2026-03-11 — После паузы `device-auth` следующим cheap production seam с нормальным ROI стал `debug_sandbox` config-loading boundary
**Решение:**
- Не идти сразу в новый тяжёлый auth/UI seam.
- Следующим шагом вынести в `gena-runtime` только config-loading helper c harness overrides для `debug_sandbox`.
- Оставить в `cli/debug_sandbox.rs`:
  - sandbox mode selection
  - sandbox-specific `ConfigOverrides`
  - сам sandbox execution path

**Причина:**
- Это маленький production-facing seam.
- Он убирает ещё один прямой вызов upstream-shaped config loader из CLI.
- Шаг compile-checkable и не тянет за собой новую широкую абстракцию.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `refactor(gena): route debug sandbox config through runtime`

**Альтернативы:**
- Игнорировать `debug_sandbox` как слишком мелкий path.
- Сразу идти в новый более тяжёлый boundary без добора этого cheap seam.

## 2026-03-11 — После headless fallback step ось `device-auth / browser-login orchestration` нужно поставить на паузу, а не продолжать ради одиночного CLI branch
**Решение:**
- Не выносить дальше CLI fallback branch из `cli/src/login.rs` в новый runtime helper.
- Считать `device-auth / browser-login` ось временно исчерпанной по cheap ROI после:
  - общего browser-login launch path
  - общего device-auth options builder
  - общего headless `request device code -> browser fallback` decision
- Следующий шаг выбирать уже вне этой auth-flow оси.

**Причина:**
- В CLI остался по сути один локальный fallback branch.
- Он уже не shared между CLI и TUI.
- Новый helper дал бы слабую пользу и начал бы размывать `gena-runtime` ради одного callsite.

**Подтверждение:**
- Residual audit выполнен после compile-green шага:
  - `9e5adcadc` — `refactor(gena): centralize headless device auth fallback`

**Альтернативы:**
- Продолжать выносить CLI fallback branch в runtime ради максимальной симметрии.
- Лезть глубже в browser/device auth state machine.

## 2026-03-11 — После общего browser-login launch path следующим safe step в `device-auth`-оси нужно брать headless fallback decision, а не тянуть дальше всю auth state machine
**Решение:**
- Не уносить дальше целиком device-code/browser orchestration.
- Следующим узким step вынести только headless transition:
  - `request_device_code(...)`
  - decision `ErrorKind::NotFound -> browser fallback`
  - возврат либо `DeviceCode`, либо уже запущенного browser-login server
- Оставить в TUI:
  - `sign_in_state` updates
  - `request_frame` scheduling
  - `block_until_done` result handling
  - `auth_manager.reload()`

**Причина:**
- Это всё ещё compile-checkable shared auth seam.
- Он уменьшает прямое знание о fallback policy в TUI headless flow.
- Более глубокий шаг уже быстро уходит в UI/state-machine territory с худшим ROI.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `9e5adcadc` — `refactor(gena): centralize headless device auth fallback`

**Альтернативы:**
- Сразу выносить deeper browser/device auth orchestration.
- Оставить `request_device_code + NotFound fallback` локально в TUI headless path.

## 2026-03-09 — В `device-auth / browser-login`-оси первым safe step нужно брать общий browser-login launch path, а не весь auth flow целиком
**Решение:**
- Не уносить сразу device-code/browser auth state machine.
- Сначала централизовать только shared browser-login launch path:
  - `run_login_server(...)`
  - start-message formatting
  - возвращаемый `auth_url/actual_port/server`
- Использовать этот helper в:
  - CLI browser login
  - TUI onboarding browser login
  - TUI headless fallback-to-browser path

**Причина:**
- Это уже shared production seam между CLI и TUI.
- Шаг compile-checkable и не требует тащить UI/state machine в runtime.
- Он даёт реальное снижение дублирования перед более тяжёлыми auth-flow шагами.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c9f2c505d` — `refactor(gena): centralize browser login launch`

**Альтернативы:**
- Сразу уносить целиком device-code/browser auth orchestration.
- Вообще не трогать shared browser-login launch и держать дублирование в CLI/TUI.

## 2026-03-09 — Residual `turn-context` helper можно было вынести, но это уже финальный low-ROI cleanup внутри `gena-runtime`
**Решение:**
- После закрытия основной stabilization-фазы всё же добрать последний маленький residual helper:
  - `build_runtime_turn_context_override(...)`
- Вынести его в отдельный модуль с сохранением `pub use` API.
- После этого считать `gena-runtime` module-splitting практически исчерпанным по ROI.

**Причина:**
- Это был последний production helper, остававшийся в корневом `lib.rs`.
- Шаг маленький, compile-checkable и завершает фасадную структуру `gena-runtime`.
- После него дальнейшие split'ы внутри `gena-runtime` уже перестают давать заметную пользу.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `4d3c43a93` — `refactor(gena): split runtime turn context module`

**Альтернативы:**
- Оставить helper в `lib.rs` и не делать даже этот final cleanup.
- Продолжать искать новые micro-split'ы внутри `gena-runtime` после этого шага.

## 2026-03-09 — После split шагов `auth/config/interaction/bootstrap/thread` фазу `gena-runtime stabilization` разумно считать практически закрытой
**Решение:**
- После выделения модулей:
  - `auth`
  - `config`
  - `interaction`
  - `bootstrap`
  - `thread`
  не продолжать дальше mechanical module-splitting `gena-runtime`.
- Оставить `gena-runtime/src/lib.rs` как тонкий фасад с `pub use` и небольшим residual surface.
- Следующий architectural step выбирать уже вне этой фазы.

**Причина:**
- `lib.rs` уже сжат примерно до фасада.
- Remaining production helper в корне практически один: `build_runtime_turn_context_override(...)`.
- Его вынос возможен технически, но это уже micro-split с низким ROI.

**Подтверждение:**
- Residual audit после commit'а:
  - `a447d0b12` — `refactor(gena): split runtime thread module`

**Альтернативы:**
- Продолжать делать микро-сплиты внутри `gena-runtime` ради полного опустошения `lib.rs`.
- Считать фазу закрытой и двигаться к следующему high-ROI boundary.

## 2026-03-09 — После `bootstrap` split следующим safe stabilization-step в `gena-runtime` нужно выносить thread/session-orchestration слой в отдельный модуль
**Решение:**
- После выделения `bootstrap`-модуля следующим модулем брать remaining thread/session-orchestration блок из `gena-runtime/src/lib.rs`.
- Вынести в отдельный модуль:
  - thread manager factory
  - default model resolution
  - start/resume thread helpers
  - fork thread helpers
- Сохранить внешний API через `pub use` из `lib.rs`.
- `build_runtime_turn_context_override(...)` пока оставить в `lib.rs` как отдельный маленький helper, чтобы не делать микро-split.

**Причина:**
- Это уже последний достаточно связный orchestration-блок в `lib.rs`.
- Он отделим от `bootstrap/provider-policy` и от interaction formatting.
- Такой split оставляет в корневом `lib.rs` только очень маленький residual surface.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `a447d0b12` — `refactor(gena): split runtime thread module`

**Альтернативы:**
- Объединять thread helpers с bootstrap в один слишком широкий модуль.
- Дробить `turn_context` в отдельный микро-step до thread split.

## 2026-03-09 — После `interaction` split следующим safe stabilization-step в `gena-runtime` нужно выносить bootstrap/provider-policy слой в отдельный модуль
**Решение:**
- После выделения `interaction`-модуля следующим модулем брать оставшийся bootstrap/provider-policy блок из `gena-runtime/src/lib.rs`.
- Вынести в отдельный модуль:
  - bootstrap context
  - OTEL/log-db bootstrap
  - OSS bootstrap/provider planning
  - trust/login/onboarding bootstrap policy
  - provider token bootstrap
- Сохранить внешний API через `pub use` из `lib.rs`.

**Причина:**
- Это следующий связный remaining блок в `lib.rs` после `auth`, `config` и `interaction`.
- Он отражает отдельную runtime responsibility: bootstrap and provider policy.
- Такой split дополнительно стабилизирует `gena-runtime` и делает следующий remaining surface заметно меньше.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `1924b2a89` — `refactor(gena): split runtime bootstrap module`

**Альтернативы:**
- Переходить сразу к thread/session split внутри `lib.rs`.
- Оставить bootstrap/provider policy размазанным в корневом `lib.rs`.

## 2026-03-09 — После `config` split следующим safe stabilization-step в `gena-runtime` нужно выносить interaction/feature слой как отдельный модуль
**Решение:**
- После выделения `config`-модуля следующим модулем брать связный interaction-блок из `gena-runtime/src/lib.rs`.
- Вынести в отдельный модуль:
  - exit formatting
  - interactive startup gating
  - clear-memory helpers
  - feature listing/enable/formatting
  - resume/fork interactive shaping
- Сохранить внешний API через `pub use` из `lib.rs`.

**Причина:**
- Это следующий плотный remaining блок в `lib.rs` после `auth` и `config`.
- Он логически связан как entrypoint-facing runtime interaction surface.
- Такой split уменьшает размер и неоднородность `lib.rs`, не трогая callsite'ы.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `c58ad8934` — `refactor(gena): split runtime interaction module`

**Альтернативы:**
- Переходить сразу к более тяжёлому bootstrap/provider split.
- Вытаскивать одиночные helper'ы по одному без явной модульной границы.

## 2026-03-09 — После TUI log DB bootstrap следующим безопасным production seam внутри auth/bootstrap зоны стал общий builder для device-auth options
## 2026-03-09 — После first auth split следующий stabilization-step в `gena-runtime` нужно делать по плотному config/session блоку, а не по случайным helper'ам
**Решение:**
- После выделения `auth`-модуля следующим safe split брать связный `config/session/config-persistence` блок из `gena-runtime/src/lib.rs`.
- Выносить его в отдельный модуль без изменения внешнего API: только через `pub use` из `lib.rs`.
- Не смешивать в этом шаге:
  - feature/output formatting,
  - interactive CLI shaping,
  - provider token / onboarding policy.

**Причина:**
- Это самый плотный remaining блок в `lib.rs` с хорошей внутренней связностью.
- Он уже содержит большой объём runtime-facing config/session/config-persist logic, который логично группировать вместе.
- Такой split уменьшает хаос в `gena-runtime` и готовит почву к следующей фазе stabilization без риска для callsite'ов.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `4b6e7c1dd` — `refactor(gena): split runtime config module`

**Альтернативы:**
- Вытаскивать маленькие несвязанные helper'ы из `lib.rs` точечно.
- Сразу идти в более тяжёлый `device-auth / browser-login orchestration`.

## 2026-03-09 — После TUI log DB bootstrap следующим безопасным production seam внутри auth/bootstrap зоны стал общий builder для device-auth options
**Решение:**
- Не идти сразу в deeper `device-auth / browser-login orchestration`.
- Сначала вынести только shared подготовку device-auth options:
  - `issuer`
  - `open_browser`
  - базовая `ServerOptions` конфигурация
- Использовать этот helper в:
  - CLI device-auth flow
  - TUI headless ChatGPT login flow

**Причина:**
- Это cheap, compile-checkable seam.
- Он реально уменьшает дублирование между CLI и TUI headless path.
- Более глубокий шаг уже быстро уходит в orchestration-heavy territory с худшим ROI.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `1302a2825` — `refactor(gena): centralize device auth options`

**Альтернативы:**
- Сразу вытаскивать deeper device-auth/browser-login orchestration.
- Вместо этого игнорировать shared option shaping и оставить дублирование.

## 2026-03-09 — После Windows sandbox persistence следующим production-facing seam с хорошим ROI оказался TUI log DB bootstrap
**Решение:**
- Следующим шагом после Windows sandbox persistence взять remaining TUI log DB bootstrap.
- Вынести только shared bootstrap helper для:
  - `state_db::get_state_db(...)`
  - `log_db::start(...)`
- Не пытаться в этом шаге уносить весь tracing registry wiring или feedback layers.

**Причина:**
- Это всё ещё production-facing observability seam.
- Он достаточно узкий и compile-checkable.
- Более широкий вынос всего subscriber wiring быстро дал бы хуже ROI и больше type-heavy glue.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `2b6edc239` — `refactor(gena): centralize tui log db bootstrap`

**Альтернативы:**
- Сразу выносить весь observability subscriber bootstrap целиком.
- Вместо этого идти в deeper device-auth/login-server flow.

## 2026-03-09 — После pause на `ChatGPT login bootstrap`-оси следующий production-facing seam с хорошим ROI: Windows sandbox persistence, но только его config-edit часть
**Решение:**
- Следующим шагом после pause на `ChatGPT login bootstrap` взять Windows sandbox persistence в `tui/app`.
- Вынести только config-edit/persistence часть:
  - `set_windows_sandbox_mode(...)`
  - `clear_legacy_windows_sandbox_keys()`
- Не выносить в этот шаг:
  - UI/state updates после apply
  - warning/open-confirmation flow
  - turn-context override side effects

**Причина:**
- Это production-facing residual seam с хорошим ROI.
- Он ложится в уже существующий runtime config-persistence слой.
- Более широкое вытаскивание всего Windows sandbox flow быстро стало бы platform/UI-heavy.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `6dea17939` — `refactor(gena): centralize windows sandbox persistence`

**Альтернативы:**
- Сразу выносить весь Windows sandbox flow целиком.
- Вместо этого идти в single-place `log_db/state_db` observability glue.

## 2026-03-09 — В `ChatGPT login bootstrap`-оси после shared `ServerOptions` builder имеет смысл ещё один CLI-local helper step, после чего ось лучше паузить
**Решение:**
- После выноса shared `ServerOptions` builder сделать ещё один узкий шаг:
  - свести CLI browser-login launch path в local helper
  - переиспользовать его в `login_with_chatgpt(...)` и browser fallback path
- После этого `ChatGPT login bootstrap`-ось временно паузить.

**Причина:**
- Внутри `cli/login.rs` всё ещё оставалось явное повторение `run_login_server(...) + print + block_until_done().await`.
- Это ещё cheap, compile-checkable cleanup.
- Дальше ось быстро уходит в TUI onboarding state machine и device/browser auth orchestration, что уже хуже по ROI.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `d4b4e0dad` — `refactor(gena): simplify cli browser login flow`

**Альтернативы:**
- Сразу тащить deeper browser/device-auth orchestration.
- Остановиться на предыдущем шаге и оставить локальное дублирование в `cli/login.rs`.

## 2026-03-09 — После cheap residual model-persistence шагов следующим boundary выбран `ChatGPT login bootstrap`, но первым срезом брать только shared `ServerOptions` builder
**Решение:**
- Следующим boundary считать `ChatGPT login bootstrap` между CLI и TUI onboarding.
- Первым шагом внутри него вынести только общий builder для `ServerOptions`.
- Не тащить в этот же шаг:
  - browser/device auth state machine,
  - `run_login_server(...)` / `block_until_done()` orchestration,
  - UI/state transitions onboarding.

**Причина:**
- `ServerOptions::new(...)` уже повторялся в нескольких продовых callsite'ах CLI и TUI.
- Это cheap, compile-checkable seam с хорошим ROI.
- Более глубокий browser/device-auth flow быстро становится UI-heavy и хуже подходит для следующего узкого шага.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `dd2f28aba` — `refactor(gena): centralize chatgpt login server options`

**Альтернативы:**
- Сразу выносить целиком login-server/device-auth orchestration.
- Вместо этого идти в single-place `log_db/state_db` observability glue.

## 2026-03-09 — После persisted auth context ещё есть смысл добрать два дешёвых residual config-persist хвоста в `tui/app`, но не уходить пока в Windows sandbox или login-server flow
**Решение:**
- Следующим шагом после `persisted auth context` добрать только два простых residual config-persist кейса в `tui/app`:
  - model availability NUX count
  - current model selection persistence
- Не заходить в этом же шаге в:
  - Windows sandbox mode persistence
  - `run_login_server(...)` / device-auth flow

**Причина:**
- Эти два кейса всё ещё были cheap, compile-checkable и хорошо ложились в уже существующий runtime config-persist слой.
- Windows sandbox path содержит больше platform-specific state changes.
- Login server/device auth flow глубже завязаны на UI/network orchestration и хуже подходят как следующий дешёвый seam.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `303804554` — `refactor(gena): centralize residual tui model persistence`

**Альтернативы:**
- Сразу идти в Windows sandbox persistence.
- Сразу идти в deeper ChatGPT login server / device auth flow.

## 2026-03-09 — После `api key login` следующим cheap shared seam является persisted auth context, а не deeper login-server flow
**Решение:**
- Не идти сразу в `run_login_server(...)` / ChatGPT device flow.
- Сначала вынести чтение persisted auth context в `gena-runtime`:
  - auth mode
  - token
  - account id
- Использовать этот helper в:
  - `cli/login status`
  - `tui/voice`

**Причина:**
- `CodexAuth::from_auth_storage(...)` всё ещё дублировался в двух user-facing entrypoints.
- Это оставалось дешёвым compile-checkable seam с хорошим ROI.
- Login server / device auth flow уже глубже завязаны на UI/network control flow и хуже подходят как следующий шаг.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `54f993f1f` — `refactor(gena): centralize persisted auth context`

**Альтернативы:**
- Сразу тащить `run_login_server(...)` или `run_device_code_login(...)` в runtime.
- Оставить persisted auth read duplicated в CLI и TUI voice path.

## 2026-03-09 — После residual audit `turn-context override`-ось можно временно считать закрытой, а следующим cheap shared seam является `api key login action`
**Решение:**
- Не продолжать `turn-context override`-ось дальше ради тестовых или low-value residual callsite'ов.
- Считать её временно закрытой после первых продовых TUI callsite'ов.
- Следующим узким shared seam взять `login_with_api_key(...)` и вынести сам action в `gena-runtime` для CLI и TUI onboarding.

**Причина:**
- В прод-коде TUI после первого шага в этой оси почти не осталось повторяющихся `OverrideTurnContext` callsite'ов.
- Дальнейшее продолжение уже давало бы низкий ROI.
- При этом `login_with_api_key(...)` всё ещё дублировался в двух user-facing entrypoints и оставался дешёвым compile-checkable seam.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `0225f5613` — `refactor(gena): centralize api key login action`

**Альтернативы:**
- Продолжать механически переводить тестовые `OverrideTurnContext` callsite'ы.
- Сразу заходить в deeper onboarding/login UX вместо узкого shared auth action.

## 2026-03-09 — После закрытия `auth/chatwidget` residuals следующим high-ROI boundary является `turn-context override`, а не новый широкий runtime-seam
**Решение:**
- Следующим concrete seam считать repeated `Op::OverrideTurnContext { ... }` callsite'ы в TUI.
- Сначала ввести:
  - нейтральный DTO в `gena-types`,
  - shared builder/helper в `gena-runtime`,
  - затем переводить на него повторяющиеся callsite'ы по одному пакету.
- Не начинать на этом шаге новый широкий слой “ещё один orchestration boundary”.

**Причина:**
- В `tui/app` и `tui/chatwidget` ещё оставалось несколько user-facing, но runtime-sensitive turn-context override payload'ов.
- Это cheap, compile-checkable seam с хорошим ROI без UI redesign.
- Нейтральный DTO даёт правильную границу и не размазывает прямой `Op::OverrideTurnContext` payload по TUI-коду.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `5eb7090d0` — `refactor(gena): introduce turn context override boundary`

**Альтернативы:**
- Сразу искать новый широкий boundary вне этой оси.
- Оставить repeated inline `Op::OverrideTurnContext` payload'ы в `tui/app` и `tui/chatwidget`.

## 2026-03-09 — После auth/cloud bootstrap следующим high-ROI boundary для `gena` является `session/thread boundary`, а не широкий `runtime orchestration layer`
## 2026-03-09 — В `ConfigEditsBuilder`-зоне после простых profiled `persist-*` операций ещё имеет смысл отдельно вынести `skill/app toggles`, но не идти дальше в feature-flags без нового ROI-аудита
## 2026-03-09 — После выноса `feature flag persistence` `ConfigEditsBuilder`-зону в `tui/app` можно считать практически закрытой и не продолжать дальше low-ROI одиночными кейсами
## 2026-03-09 — После закрытия `ConfigEditsBuilder`-оси следующий high-ROI boundary смещается в `cli/mcp_cmd` global MCP server persistence
## 2026-03-09 — После выноса MCP persistence и config loading `mcp_cmd` можно временно оставить: дальше там в основном OAuth/UI territory с более низким ROI
## 2026-03-09 — После закрытия `MCP`-оси следующий дешёвый и полезный seam: auth/config loading в отдельных entrypoints, без захода в deeper auth internals
## 2026-03-09 — После дешёвых auth/config seams следующий high-ROI boundary смещается в общий observability/bootstrap path `exec` + `tui`
## 2026-03-09 — После observability шага следующий cheap auth seam: общий logout action для CLI и TUI
## 2026-03-09 — После logout seam и chatwidget home residual step auth/home action zone можно временно считать закрытой
**Решение:**
- После:
  - общего logout action,
  - маленького residual step с `chatwidget` theme-picker home lookup,
не продолжать дальше механически резать эту зону.
- Следующий boundary искать уже вне `auth-action/chatwidget` оси.

**Причина:**
- Главный общий auth action (`logout`) уже вынесен.
- Оставшийся `chatwidget` home lookup был дешёвым residual step и теперь тоже убран.
- Дальше зона начинает упираться в deeper login/onboarding UX и даёт меньший ROI.

**Подтверждение:**
- Финальный residual step этой оси:
  - `9354e4b7c` — `refactor(gena): route chatwidget home lookup through adapter`

**Альтернативы:**
- Продолжать механически резать auth/onboarding UI internals.
- Оставить direct `find_codex_home()` residual callsite в `chatwidget`.

## 2026-03-09 — После observability шага следующий cheap auth seam: общий logout action для CLI и TUI
**Решение:**
- Не заходить сразу в deeper auth internals.
- Сначала вынести общий logout action в Gena-owned runtime layer и использовать его и в CLI, и в TUI.
- API key login / OAuth / onboarding login flows оценивать отдельно уже после этого узкого шага.

**Причина:**
- `logout(...)` оставался duplicated user-facing auth action в двух entrypoints.
- Это дешёвый compile-checkable seam без UI redesign.
- Он снижает direct dependence entrypoints на `codex_core::auth` хотя бы в одной общей операции.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `f47a540b7` — `refactor(gena): centralize logout auth action`

**Альтернативы:**
- Сразу идти в `login_with_api_key(...)` и deeper login flows.
- Оставить logout duplicated в CLI и TUI.

## 2026-03-09 — После дешёвых auth/config seams следующий high-ROI boundary смещается в общий observability/bootstrap path `exec` + `tui`
**Решение:**
- Следующим concrete seam после `auth/config loading` считать общий observability/bootstrap path в `exec` и `tui`.
- Первым шагом внутри него выносить именно safe OTEL provider bootstrap:
  - panic-safe wrapper
  - error shaping
  - `Option<OtelProvider>` construction
- `log_db` и остальной telemetry glue оценивать отдельно уже после этого шага.

**Причина:**
- Этот bootstrap уже дублировался в двух entrypoints.
- Он остаётся upstream-sensitive и даёт реальный выигрыш для sync, не затрагивая UI/domain logic.
- Такой разрез остаётся cheap, compile-checkable и не требует тяжёлой generic-магии.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `6662bd2b3` — `refactor(gena): centralize observability bootstrap`

**Альтернативы:**
- Сразу тащить весь `log_db`/telemetry glue вместе с OTEL.
- Игнорировать shared observability bootstrap и идти в менее конкретную зону.

## 2026-03-09 — После дешёвых auth/config seams следующий high-ROI boundary смещается в общий observability/bootstrap path `exec` + `tui`
**Решение:**
- После `mcp_cmd` не лезть сразу в OAuth internals.
- Сначала вынести оставшийся cheap entrypoint glue вокруг:
  - config loading в `cli/login.rs`
  - config loading/home resolution в `tui/voice.rs`
- Не трогать в этом шаге deeper auth logic, login UX и onboarding UI.

**Причина:**
- Это дешёвый compile-checkable seam с хорошим ROI.
- Он ещё уменьшает прямую зависимость entrypoints от `codex_core` bootstrap/config loading.
- При этом не требует premature redesign auth flows.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `4035a16a5` — `refactor(gena): route auth config loading through runtime`

**Альтернативы:**
- Сразу идти в deeper OAuth/login/onboarding internals.
- Оставить локальный config/home loading в `login.rs` и `voice.rs`.

## 2026-03-09 — После выноса MCP persistence и config loading `mcp_cmd` можно временно оставить: дальше там в основном OAuth/UI territory с более низким ROI
**Решение:**
- После выноса из `mcp_cmd`:
  - global MCP server persistence,
  - config loading,
  - product home resolution,
не продолжать автоматически глубже в `mcp_cmd`.
- Следующий boundary выбирать уже вне `MCP`-оси.

**Причина:**
- Оставшаяся логика в `mcp_cmd` больше относится к:
  - OAuth login/logout interaction,
  - transport probing,
  - user-facing list/get formatting.
- Это менее выгодно для upstream sync, чем уже вынесенные persistence/config seams.
- Продолжение тут дало бы хуже ROI, чем новый concrete boundary в другой зоне.

**Подтверждение:**
- Финальный рациональный шаг текущей `MCP`-оси:
  - `e984bf88f` — `refactor(gena): route mcp config loading through runtime`

**Альтернативы:**
- Продолжать дальше вытаскивать OAuth/login flow из `mcp_cmd`.
- Продолжать смешивать CLI formatting и runtime concerns в той же оси.

## 2026-03-09 — После закрытия `ConfigEditsBuilder`-оси следующий high-ROI boundary смещается в `cli/mcp_cmd` global MCP server persistence
**Решение:**
- Следующим concrete seam после `ConfigEditsBuilder`-оси считать `MCP config persistence` в `cli/mcp_cmd`.
- Первым срезом внутри него выносить только global MCP server config persistence:
  - add global server
  - remove global server
- Не тащить в этот же шаг:
  - OAuth login/logout flow
  - transport probing
  - user-facing CLI copy

**Причина:**
- Это повторяющийся upstream-sensitive config-edit glue в user-facing CLI.
- Он даёт более чистый и дешёвый шаг, чем сразу лезть в весь `mcp_cmd` целиком.
- Такой разрез сохраняет чистую границу: runtime отвечает за persistence, CLI — за validation, transport shaping и OAuth interaction.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `7beaf621e` — `refactor(gena): centralize mcp config persistence`

**Альтернативы:**
- Сразу уносить весь `mcp_cmd` вместе с OAuth/login logic.
- Игнорировать `mcp_cmd` и идти в менее конкретный широкий boundary.

## 2026-03-09 — После выноса `feature flag persistence` `ConfigEditsBuilder`-зону в `tui/app` можно считать практически закрытой и не продолжать дальше low-ROI одиночными кейсами
**Решение:**
- После выноса:
  - простых profiled `persist-*` операций,
  - `skill/app toggles`,
  - `UpdateFeatureFlags` persistence,
считать `ConfigEditsBuilder`-ось в `tui/app` практически закрытой.
- Не продолжать автоматически выносить:
  - startup NUX counter persist,
  - одиночный current model persist path,
  - Windows-specific sandbox config path
без нового отдельного ROI-обоснования.

**Причина:**
- Основной повторяющийся config-persistence glue уже вынесен.
- Оставшиеся кейсы либо одиночные, либо platform-specific, либо дают слишком малый выигрыш для upstream sync.
- Продолжение этой оси теперь скорее размоет границу и даст меньше пользы, чем переход к новому architectural boundary.

**Подтверждение:**
- Последний рациональный шаг этой оси:
  - `cd5243924` — `refactor(gena): centralize tui feature flag persistence`

**Альтернативы:**
- Продолжать mechanically резать оставшиеся одиночные `ConfigEditsBuilder` callsites.
- Смешивать low-ROI persistence cleanup с unrelated UI/platform-specific flows.

## 2026-03-09 — В `ConfigEditsBuilder`-зоне после простых profiled `persist-*` операций ещё имеет смысл отдельно вынести `skill/app toggles`, но не идти дальше в feature-flags без нового ROI-аудита
**Решение:**
- После выноса простых profiled `persist-*` операций сделать ещё один узкий runtime-boundary step для:
  - `SetSkillEnabled`
  - `SetAppEnabled`
- Не продолжать автоматически дальше в:
  - `UpdateFeatureFlags`
  - Windows-specific config toggle paths
  - одиночные низкоценные persist-кейсы
без отдельного residual audit.

**Причина:**
- `skill/app toggles` всё ещё представляют собой повторяющийся config-persist glue с хорошим ROI.
- При этом `feature flags` уже содержат больше доменной логики и platform-specific side effects, поэтому их вынос дороже и менее очевиден.
- Такой порядок сохраняет clean runtime/config boundary и не тащит в `gena-runtime` лишнюю UI/domain orchestration.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `40c9cfa07` — `refactor(gena): centralize skill and app config toggles`

**Альтернативы:**
- Сразу идти в `UpdateFeatureFlags` и Windows-specific flows.
- Оставить `skill/app toggles` локально в `tui/app`, несмотря на повторяющийся config persistence glue.

## 2026-03-09 — После auth/cloud bootstrap следующим high-ROI boundary для `gena` является `session/thread boundary`, а не широкий `runtime orchestration layer`
**Решение:**
- Следующим архитектурным этапом после `auth/cloud bootstrap` считать `session/thread boundary`.
- Первым срезом внутри него выносить в Gena-owned слои:
  - session target lookup по `id/name`
  - latest session lookup
  - thread id resolution из rollout/meta
  - session cwd metadata path
  - cwd comparison policy
- После этого уже отдельно переходить к `ThreadManager / session bootstrap boundary`.

**Причина:**
- Эта зона остаётся upstream-sensitive и была размазана по `codex-exec` и `codex-tui`.
- Она даёт более конкретный и проверяемый seam, чем абстрактный широкий `runtime orchestration layer`.
- Вынос этого слоя сразу снижает direct knowledge entrypoints о thread/session internals и готовит почву для следующего bootstrap-среза.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `dd06815f5` — `refactor(gena): extract session and thread lookup boundary`

**Альтернативы:**
- Сразу идти в широкий `runtime orchestration layer` (слишком расплывчатая граница).
- Рано переходить к `plugin-api` (меньший практический ROI на текущем этапе).

## 2026-03-09 — После lookup-слоя следующий срез `session/thread boundary` должен идти через общий `ThreadManager / thread bootstrap`, а не через новый абстрактный orchestration-layer
**Решение:**
- После выноса lookup/meta helper'ов следующим шагом держать в `gena-runtime` общий bootstrap для:
  - `ThreadManager` factory
  - default model resolution
  - start/resume thread bootstrap
  - fork thread bootstrap
- Использовать этот слой сразу и в `codex-exec`, и в `codex-tui/src/app.rs`.

**Причина:**
- Именно этот узел оставался плотным upstream-sensitive glue и повторялся в двух entrypoints.
- Такой срез остаётся конкретным, compile-checkable и не требует premature wide abstraction.
- После него можно уже оценивать остаточный thread/session knowledge в `tui/app` точечно.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `3878d263d` — `refactor(gena): move thread bootstrap into runtime`

**Альтернативы:**
- Перейти сразу к широкому runtime orchestration layer без следующего concrete seam.
- Оставить `ThreadManager` bootstrap размазанным в `exec` и `tui/app`.

## 2026-03-09 — После вынесения `ThreadManager` bootstrap в runtime стоит также убирать in-app session switching из `tui/app`, если это не UI, а thread-operation glue
**Решение:**
- Если ветка `tui/app` выполняет:
  - resume existing session,
  - fork current session,
и в ней смешаны UI-parts и direct thread operations, то direct thread operations тоже уносить в `gena-runtime`.
- В `tui/app` оставлять:
  - picker / prompt / cwd rebuild policy
  - UI re-init
  - history/status rendering

**Причина:**
- Это продолжение той же `session/thread boundary`, а не новый отдельный домен.
- Direct thread operations внутри `tui/app` остаются upstream-sensitive glue даже после выноса bootstrap.
- Такой шаг всё ещё даёт хороший ROI, пока не начинает ломать UI/runtime границу.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `7c3f7b295` — `refactor(gena): route tui session switching through runtime`

**Альтернативы:**
- Оставить in-app resume/fork operations локально в `tui/app`.
- Продолжать резать `tui/app` дальше без residual audit и оценки diminishing returns.

## 2026-03-09 — После паузы на `session/thread boundary` следующий high-ROI шаг смещается в `tui app config mutation / rebuild`, а не в дальнейший thread-thinning
**Решение:**
- После того как `lookup`, `thread bootstrap` и `in-app session switching` уже вынесены, не продолжать дальше механически резать `session/thread`.
- Следующим шагом выносить в `gena-runtime` config-mutation helpers из `tui/app`, прежде всего:
  - config rebuild for cwd
  - refresh-from-disk path
  - carry-forward approval/sandbox policy overrides

**Причина:**
- Это остаётся upstream-sensitive glue и напрямую влияет на merge-cost.
- В отличие от остаточного `thread/session` кода, здесь ещё много прямого знания о config internals.
- Такой шаг сохраняет чистую границу: runtime отвечает за config mutation policy, `tui/app` — за UI side effects и error presentation.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `44fdb000c` — `refactor(gena): move tui config mutation helpers into runtime`

**Альтернативы:**
- Продолжать дальше резать `session/thread` при уже падающем ROI.
- Сразу идти в широкий `plugin-api` или другой абстрактный слой без следующего concrete seam.

## 2026-03-09 — Внутри `ConfigEditsBuilder`-зоны сначала стоит выносить простые profiled `persist-*` операции, а не feature/app toggles
**Решение:**
- После выноса config rebuild / refresh helpers следующим узким шагом выносить в `gena-runtime` shared helper для profiled `ConfigEditsBuilder`-chains.
- На него переводить сначала операции вида:
  - `set_model`
  - `set_personality`
  - `set_realtime_*`
  - notice acknowledgement flags
  - model migration acknowledgement
- Не начинать с feature/app toggles и Windows-specific путей.

**Причина:**
- Это самый повторяющийся и низкорисковый слой config persistence.
- Он даёт хороший ROI без захвата более доменных веток `tui/app`.
- После него можно отдельно переоценить, остался ли ещё один выгодный срез в `ConfigEditsBuilder`-зоне.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `543beeb87` — `refactor(gena): centralize profiled tui config edits`

**Альтернативы:**
- Сразу идти в feature/app toggles (больше доменной логики, выше риск overreach).
- Оставить repeated profiled builder chains локально в `tui/app`.

## 2026-03-09 — После runtime extraction продолжать чистить `codex-cli` через dispatch helper-ы, даже если это не требует новых crates
**Решение:**
- Не ограничивать архитектурную очистку только переносами между crates.
- После появления основных Gena-owned boundaries продолжать thinning `codex-cli` за счет малых dispatch cleanup шагов внутри самого entrypoint.
- Допустимый паттерн:
  - вынос repeated glue в локальные helper’ы,
  - уменьшение nested flow внутри `match subcommand`,
  - сокращение копипасты вокруг `prepend_config_flags(...)`, `run_main(...)`, `handle_app_exit(...)`.

**Причина:**
- Основной источник diff к upstream всё ещё сидит в `cli/src/main.rs`.
- Даже если новый helper не создаёт отдельный crate boundary, он делает entrypoint тоньше и снижает шум последующих архитектурных шагов.
- Малые compile-green dispatch cleanup шаги дешёвые, безопасные и хорошо подходят для живого форка.

**Подтверждение:**
- В `codex-rs/chore/update-arch` запушен commit:
  - `a92986daa` — `refactor(gena): simplify cli subcommand dispatch`
- После этого проверки прошли:
  - `cargo fmt --all`
  - `cargo check -p codex-cli -j1` PASS

**Альтернативы:**
- Оставлять repeated dispatch glue в `cli`, пока не появится повод для большого crate-level refactor.
- Пытаться любой cleanup делать только через новые абстракции/workspace crates, даже когда достаточно локального helper extraction.

## 2026-03-09 — Для `handle_app_exit` выносить orchestration в runtime-plan, но оставлять `UpdateAction` execution в `codex-cli`
**Решение:**
- После выноса exit formatting не переносить в Gena-owned слой реальное исполнение `UpdateAction`.
- Вместо этого вынести ещё один уровень orchestration:
  - `RuntimeExitPlan`
  - `plan_runtime_exit(...)`
- В `codex-cli` оставить:
  - `std::process::exit(...)`
  - platform-specific command execution
  - `wsl_paths` / `#[cfg(windows)]` ветвление для update shell

**Причина:**
- `handle_app_exit` действительно можно делать тоньше, но `run_update_action` упирается в платформенные детали запуска команды.
- Тащить эти детали в `gena-runtime` означало бы либо загрязнять runtime shell-specific знанием, либо размножать platform branches вне entrypoint.
- Runtime-plan сохраняет правильную границу:
  - orchestration и user-facing semantics в Gena-owned слое,
  - platform execution detail в `cli`.

**Подтверждение:**
- В `codex-rs/chore/update-arch` запушены шаги:
  - `f4e7cd11c` — `refactor(gena): move update messaging into branding`
  - `79c561928` — `refactor(gena): route cli exit handling through runtime plan`
- Проверки:
  - `cargo check -p gena-branding -p codex-cli -j1` PASS
  - `cargo check -p gena-runtime -p codex-cli -j1` PASS

**Альтернативы:**
- Оставить весь `handle_app_exit` в `codex-cli` как есть.
- Пытаться перенести `UpdateAction` execution глубже в runtime и смешивать там platform-specific shell behavior.

## 2026-03-09 — После основных runtime extraction добивать `cli` малыми compile-green срезами: formatting и command wiring тоже считать boundary logic
**Решение:**
- После выноса крупных кусков (`interactive`, `startup`, `exit`, `state`) продолжать thinning `codex-cli` не только большими переносами, но и малыми срезами.
- Считать полезными для выноса также:
  - feature output formatting,
  - root command wiring / CLI identity builder.
- Для этого:
  - feature message/table formatting переносить в `gena-runtime`,
  - root CLI identity держать в `gena-branding`.

**Причина:**
- Даже небольшие user-facing formatting/wiring блоки продолжают размазывать Gena-owned logic по entrypoint.
- Малые compile-green шаги дешевле по риску и позволяют чистить `cli` без premature abstraction и без открытия новых dependency cycles.
- Такой режим лучше подходит для живого форка, чем редкие большие refactor-пакеты.

**Подтверждение:**
- В `codex-rs/chore/update-arch` запушен commit:
  - `501c0d7ba` — `refactor(gena): reduce cli surface for features and branding`
- После этого проверки прошли:
  - `cargo fmt --all`
  - `cargo check -p gena-branding -p codex-cli -j1` PASS

**Альтернативы:**
- Оставить formatting/wiring в `codex-cli`, считая это слишком мелким для архитектурной работы.
- Ждать только крупных переносов и копить мелкий boundary drift в entrypoint.

## 2026-03-09 — Для дальнейшего thinning `cli` выносить exit/startup logic через shared DTO и runtime-preparation, а не оставлять formatting в entrypoint
**Решение:**
- После выноса feature/memory/interactive shaping продолжить thinning `codex-cli` через перенос в `gena-runtime` ещё двух зон:
  - exit message / fatal-exit shaping
  - interactive startup preparation и dumb-terminal gating decision
- Для exit path ввести в `gena-types` отдельный shared DTO:
  - `GenaAppExitInfo`
  - `GenaExitReason`
- В `codex-cli` оставить только:
  - conversion shell (`AppExitInfo -> GenaAppExitInfo`, `TuiCli <-> GenaInteractiveCli`)
  - реальный terminal IO (`confirm`)
  - запуск `codex_tui::run_main(...)`

**Причина:**
- Entry-point не должен держать runtime-facing message formatting и decision logic, если цель ветки — уменьшить upstream-sensitive knowledge в `cli`.
- При этом тащить `codex_tui`-типы напрямую в `gena-runtime` всё ещё нельзя из-за риска снова открыть грязные зависимости или цикл.
- Shared DTO + runtime preparation сохраняют dependency direction и продолжают additive migration без большого rewrite.

**Подтверждение:**
- В `codex-rs/chore/update-arch` запушен commit:
  - `c69e8c288` — `refactor(gena): move cli exit and startup flows into runtime`
- После этого проверки прошли:
  - `cargo fmt --all`
  - `cargo check -p gena-types -p gena-runtime -p codex-cli -j1` PASS

**Альтернативы:**
- Оставить exit/startup formatting и gating decision в `codex-cli`.
- Пытаться тащить `codex_tui` runtime-knowledge напрямую в `gena-runtime` вместо neutral/shared boundary.

## 2026-03-09 — Для переноса interactive CLI orchestration использовать нейтральные DTO в `gena-types`, а не зависимость `gena-runtime -> codex-tui`
**Решение:**
- При выносе interactive resume/fork shaping из `codex-cli` не делать прямую зависимость `gena-runtime` от `codex-tui::Cli`.
- Вынести нейтральный shared DTO в `gena-types`:
  - `GenaInteractiveCli`
- Держать orchestration-функции над этим DTO в `gena-runtime`, а конвертацию между `codex_tui::Cli` и `GenaInteractiveCli` оставлять в `codex-cli`.

**Причина:**
- Попытка прямого переноса через импорт `codex-tui::Cli` в `gena-runtime` создала cycle dependency:
  - `tui -> gena-runtime -> codex-tui`
- Такой цикл ломает чистое направление зависимостей и мешает дальше выносить entrypoint-knowledge из `cli`.
- Нейтральный DTO сохраняет dependency direction:
  - `gena-types` как shared boundary,
  - `gena-runtime` как orchestration layer,
  - `codex-cli` как thin conversion shell.

**Подтверждение:**
- В `codex-rs/chore/update-arch` запушен commit:
  - `190fcec0b` — `refactor(gena): move interactive cli shaping behind neutral dto`
- После этого проверки прошли:
  - `cargo check -p gena-types -p gena-runtime -p codex-cli -j1` PASS

**Альтернативы:**
- Оставить interactive shaping в `codex-cli` и не выносить его в `gena-runtime`.
- Разрешить `gena-runtime` зависеть от `codex-tui` и принять cycle/грязное направление зависимостей.

## 2026-03-09 — Архитектурный rewrite `gena` вести через additive Gena-owned layers, а не через big-bang перестройку repo
**Решение:**
- Для переписывания архитектуры `gena` не делать немедленный полный перенос `codex-rs` в новый идеальный workspace layout.
- Сохранять upstream-owned часть репозитория максимально близкой к Codex upstream.
- Новые Gena-specific границы вводить постепенно отдельными crate-слоями:
  - `gena-branding`
  - `gena-config`
  - `gena-upstream-adapter`
  - затем `gena-types` / `gena-runtime` / plugin boundaries
- Главный приоритет первых шагов: уменьшать площадь diff к upstream и локализовать future breakage внутри adapter/config/branding слоев.

**Причина:**
- Полный filesystem rewrite или массовый перенос upstream crates резко удорожает merge с апстримом и создает лишний шум в истории.
- Основная цель форка `gena` сейчас не “красивое дерево”, а дешевый upstream sync и изоляция Gena-brand/Gena-features.
- Практически это уже подтвердилось на первом срезе: `gena-branding`, `gena-config` и `gena-upstream-adapter` удалось встроить без поломки текущего repo layout и без остановки сборки.

**Подтверждение:**
- Архитектурный документ в Obsidian: `GENA_REPO_STRUCTURE.md` обновлен под реалистичный migration path.
- В `codex-rs/chore/update-arch` запушен commit:
  - `0e540dcc8` — `refactor(gena): introduce branding config and upstream adapter layers`
- Проверки compile-green для архитектурного шага:
  - `cargo check -p gena-branding -p gena-config -p gena-upstream-adapter -p codex-core -p codex-cli -p codex-tui -p codex-utils-home-dir -j1` PASS
  - `cargo check -p gena-upstream-adapter -p codex-exec -p codex-tui -p codex-cli -j1` PASS

**Альтернативы:**
- Сразу переделать repo под полный target layout из `GENA_REPO_STRUCTURE.md` (слишком дорого и конфликтно для живого форка).
- Продолжать размазывать Gena-specific код по upstream crates (упрощает short-term правки, но ухудшает future sync с Codex).

## 2026-03-09 — После `branding/config/adapter/types/runtime` следующим приоритетом держать очистку `cli`/entrypoint boundaries, а не ранний `plugin-api`
**Решение:**
- После введения слоев `gena-branding`, `gena-config`, `gena-upstream-adapter`, `gena-types`, `gena-runtime` не переходить сразу к `gena-plugin-api`.
- Следующим приоритетом оставить дальнейшее срезание прямого runtime/bootstrap/state knowledge из entrypoints, в первую очередь из `codex-cli`.
- `plugin-api` начинать только после того, как boundary `entrypoints -> gena-runtime -> gena-upstream-adapter -> upstream` станет заметно чище и стабильнее.

**Причина:**
- Главная цель текущего этапа — удешевить upstream sync и уменьшить diff к Codex upstream.
- Этого сейчас сильнее достигает вынос knowledge из `cli` / `tui` / `exec`, чем раннее введение plugin abstractions.
- Слишком ранний `plugin-api` создает риск “архитектуры наперед”: новые интерфейсы появятся раньше, чем стабилизируются runtime boundaries, и потом их придется ломать.

**Подтверждение:**
- Уже введены и используются compile-green слои:
  - `gena-types` (`bd7c7157e`)
  - `gena-runtime` bootstrap layer (`79bfebfd9`)
  - runtime config path в `cli` routed через `gena-runtime` (`8f7a178d9`)
- После этого следующим полезным шагом стал ещё один runtime extraction:
  - state/memory reset helper из `codex-cli` вынесен в `gena-runtime`
- Проверки для этих шагов прошли:
  - `cargo check -p gena-runtime -p codex-exec -p codex-tui -j1` PASS
  - `cargo check -p gena-runtime -p codex-cli -p codex-exec -p codex-tui -j1` PASS

**Альтернативы:**
- Сразу вводить `gena-plugin-api` и plugin contracts до стабилизации runtime/entrypoint boundaries.
- Оставить `cli` как место, где живут orchestration и state-reset знания, и двигаться дальше по новым crate names без реального уменьшения coupling.

## 2026-03-08 — Installer default for macOS: npm global bin first
**Решение:**
- В self-extract installer для `gena` использовать приоритет установки в npm global bin (`$(npm prefix -g)/bin`) как дефолтный путь.
- Если `npm` отсутствует на целевой машине, installer пытается bootstrap (macOS: `brew install node`; Debian/Ubuntu: `apt-get install npm`), затем повторно выбирает npm global bin.
- При невозможности использовать npm global bin применять fallback (`/usr/local/bin` или `~/.local/bin`).

**Причина:**
- Пользователь ожидает UX уровня npm global install: команда `gena` должна запускаться из любой директории без ручного PATH шага.
- Это снижает friction первого запуска на новых машинах.

**Альтернативы:**
- Оставить дефолт только `~/.local/bin` и требовать `source ~/.zprofile` после установки.
- Вынести npm global bin в отдельный optional флаг installer (дополнительная сложность для пользователя).

## 2026-03-08 — Gena macOS distribution workflow (build once, install without Rust)
**Решение:**
- Зафиксировать единый flow для передачи сборок `gena` на macOS:
  - на машине сборки:
    - `CARGO_BUILD_JOBS=1 scripts/package-gena-linux-macos.sh release`
    - `scripts/make-gena-self-extract.sh dist/gena-v<version>-macos-arm64.tar.gz`
  - передавать коллеге один файл:
    - `dist/gena-v<version>-macos-arm64-installer.sh`
  - у коллеги:
    - `chmod +x gena-v<version>-macos-arm64-installer.sh`
    - `./gena-v<version>-macos-arm64-installer.sh`
    - проверка `~/.local/bin/gena --version`.
- Подробную инструкцию держать в `codex-rs/docs/distribution-linux-macos.md`, в `README.md` оставить короткий quick-flow + ссылку.

**Причина:**
- Коллегам не нужен Rust/toolchain для запуска prebuilt-бандла.
- One-file installer упрощает передачу и снижает количество ошибок при ручной установке.

**Альтернативы:**
- Передавать только `tar.gz + sha256` (надежно, но больше ручных шагов у получателя).
- Требовать локальную сборку на каждой машине (увеличивает порог входа и время запуска).

## 2026-02-28 — `gena-tui` logo baseline: default `natural`, advanced processing behind explicit variant
**Решение:**
- Для текущего UX baseline зафиксировать дефолтный рендер лого как `GENA_TUI_LOGO_VARIANT=natural` (без обязательных спец-оверлеев).
- Детализирующую обработку оставить отдельным явно включаемым вариантом (`natural_detail`) через env-переключатель.
- Не включать в baseline агрессивный `strict` landmark overlay, так как он ухудшает визуальное соответствие оригинальному арту.

**Причина:**
- Пользовательская проверка показала, что `strict`-символы заметно искажают картинку.
- `natural` даёт лучший общий баланс узнаваемости/чистоты в текущем размере логотипа.
- Вариант `natural_detail` нужен как безопасная зона для дальнейших экспериментов без деградации дефолтного UX.

**Альтернативы:**
- Сделать `natural_detail` дефолтом сразу (риски визуальной нестабильности между терминалами).
- Сохранять только один вариант без переключателей (сложнее итерировать и откатывать).

## 2026-02-17 — Ship-процесс codex binaries: version-aware launchers из download/agents
**Решение:**
- Фиксируем для Android/Termux схему поставки: build/ship кладет бинарники в `/root/download/agents`, запуск выполняется через launchers в `/usr/local/bin` (копия в `/tmp` перед exec).
- Launchers должны поддерживать оба режима:
  - `latest` (неверсионированные имена `aqa-codex`, `aqa-codex-tui`),
  - явная версия (`aqa-codex-<version>`, `aqa-codex-tui-<version>`).
- В ship-контуре закрепляем Android-safe defaults и wake-lock как стандарт для долгих сборок.

**Причина:**
- Каталог `download/agents` в текущем окружении нестабилен как exec-точка (права/mount-поведение), из-за чего прямой запуск бинаря часто ломается.
- Нужен предсказуемый запуск нескольких версий без ручного копирования и без конфликта с системным `codex`.

**Альтернативы:**
- Запускать бинарь напрямую из `download/agents` (ненадежно в Android/Termux).
- Держать только один `latest` бинарь без поддержки фиксированной версии.

## 2026-02-11 — Loop-архитектура sandbox: orchestrator-first + session persistence
**Решение:**
- Для sandbox `sb-cli-agent-loop` закрепить loop-архитектуру с явным разделением фаз (`Planner/Actor/Checker`) и централизованным tool orchestration слоем для `approval/sandbox/retry`.
- Не смешивать управление base-agent и loop в chat:
  - `/agent on|off` — управление base project-agent.
  - `/agent loop on|off` — включение/выключение loop для agent path.
- Ввести file-based persistence loop-состояния:
  - snapshot контекста (`<sessionId>.json`);
  - trace событий (`<sessionId>.trace.jsonl`);
  - конфигурация через `SB_LOOP_SESSION_ID`, `SB_LOOP_STATE_DIR`.
- Закрепить обязательный кандидатный gate перед выдачей артефакта:
  - `build` + `unit` + `bin check` + `pack` + `isolated install` + `smoke`.
- Добавить policy-guards в loop core и человекочитаемый trace-view как обязательные элементы baseline-loop:
  - guards: `maxReplans`, `maxToolCalls`, phase timeouts;
  - trace-view: `trace list` / `trace show <sessionId>`;
  - подтверждение через e2e multi-cycle сценарий `ask_user -> approve -> done`.

**Причина:**
- Снизить отклонение от принципов loop в Codex (единый orchestration policy-слой и событийная трассировка).
- Избежать регрессий существующего `fs/project-agent` поведения при добавлении loop.
- Обеспечить воспроизводимый и проверяемый side-release процесс перед публикацией `.tgz`.

**Альтернативы:**
- Оставить approvals/retry внутри отдельных tool-адаптеров и actor (хуже управляемость policy).
- Полагаться только на in-memory состояние loop без `sessionId` и trace-файлов.
- Выпускать side-артефакты без единообразного candidate gate.

## 2026-02-08 — Явный root для side-run + детерминированный FS ответ
**Решение:**
- Для `fs-agent`/`project-agent` закрепить явный приоритет root:
  1) `AQA_REPO_ROOT`, 2) `INIT_CWD`, 3) `cwd`.
- На запросы вне root возвращать явный `FS_SCOPE_ERROR`, а не «лучшее предположение».
- Для запроса несуществующей папки возвращать `PATH_NOT_FOUND`.
- Для `pwd`/«текущий путь» возвращать `repo_root` детерминированно (без прохода через LLM-loop).

**Причина:**
- В side-run режиме (запуск из `/root/download/agents`) без явного root агент видел только директорию с `.tgz`.
- Пользовательские запросы с путями должны иметь предсказуемый и проверяемый результат.
- Уменьшаем количество «тихих» ложноположительных ответов от модели.

**Альтернативы:**
- Оставить выбор root только через `cwd`/`INIT_CWD` и обучать пользователя вручную.
- Не добавлять precheck `pwd`/folder и полагаться на tool-выбор LLM.

## 2026-02-08 — Sandbox project-agent: core reuse + default agent mode
**Решение:**
- Переиспользовать логику FS/Project Agent из `@aqa/core` в sandbox-пакете `@aqa/sb-cli-llmops-project-agent` через тонкие адаптеры, а не поддерживать отдельную параллельную реализацию.
- Для project-agent закрепить tool-first паттерн: сначала `grep` по проекту, затем `read` нужных фрагментов (`offset/limit`).
- В интерактивном `chat` включить режим агента по умолчанию (`/agent on` default), чтобы пользователю не требовалось ручное переключение.
- Поставлять sandbox-архив как self-contained `.tgz` с bundled `@aqa/core` для установки без зависимости от внешнего npm registry.

**Причина:**
- Снижаем дублирование и расхождение поведения между sandbox и основной логикой.
- Ускоряем и стабилизируем анализ проекта (поиск + slice-чтение вместо больших файлов целиком).
- Убираем частую пользовательскую ошибку "агент не видит ФС" при забытом `/agent on`.
- Избегаем ошибок установки типа `Cannot find module '@aqa/core'` и `E404 @aqa/core`.

**Альтернативы:**
- Оставить отдельную sandbox-реализацию инструментов (дороже в поддержке).
- Сохранять `/agent off` по умолчанию и полагаться на обучение пользователя.
- Публиковать `@aqa/core` во внешний registry как обязательное условие установки sandbox-пакета.

## 2026-02-07 — Продакшенизация UX из sandbox
**Решение:**
- UX доработки сначала отрабатываются в `sb-agent-ux`, затем переносятся в основной `@aqa/agent-cli` без изменения имени боевого бинаря (`aqa-agent`).
- Для UI/UX-изменений основного CLI повышать patch-версию пакета (с `0.0.1` до `0.0.2` в текущем цикле).
- Артефакты упаковки (`*.tgz`) не хранятся в git.

**Причина:**
- Сохраняем стабильный путь запуска пользователя (`aqa-agent`) и предсказуемый релизный цикл.
- Отделяем исходники от бинарных артефактов и уменьшаем шум в истории git.

**Альтернативы:**
- Менять имя боевого CLI при каждом UX-эксперименте.
- Коммитить `.tgz` в репозиторий для ручной дистрибуции.

## 2026-02-03 — First‑run env и упаковка CLI
**Решение:**
- CLI при первом запуске запрашивает `LLMOPS_TOKEN` и сохраняет его в `~/.aqa-agent/.env`.
- Логи `dotenv` по умолчанию тихие; включаются через `DOTENV_CONFIG_QUIET=off`.
- Упаковка CLI в self‑contained `.tgz` выполняется скриптами `packages/infra/scripts/pack-agent-cli.sh` и `pack-sandbox.sh`.

**Причина:**
- Упростить первичную настройку пользователя и скрыть служебные логи.
- Дать воспроизводимую поставку CLI без исходников.

**Альтернативы:**
- Хранить токен только в `.env` репозитория.
- Использовать глобальные переменные окружения без файла.

## 2026-02-18 — Валидация llmops-модели перед фиксацией в aqa-codex
**Решение:**
- Перед фиксацией модели в `~/.aqa-codex/config.toml` использовать проверку `scripts/check-llmops-tools.sh` и выбирать только модель, которая возвращает нативный `tool_calls` в OpenAI-совместимом формате `chat/completions`.
- Модели, которые отдают псевдо-инструкции (`<think>`, `<tool_call>`) текстом, считать несовместимыми для production-профиля `aqa-codex`.

**Причина:**
- TUI/CLI ожидают структурированные tool-вызовы. Текстовые псевдо-теги не исполняются как инструменты и ухудшают UX/надежность.

**Альтернативы:**
- Фильтровать и парсить псевдо-теги на стороне клиента как временный workaround.
- Оставить модель без tool-calling и использовать только в plain-chat задачах.

## 2026-02-18 — Исключение локального Obsidian metadata из git
**Решение:**
- Добавить `.obsidian/` в `.gitignore` Obsidian-репозитория.

**Причина:**
- Локальные UI/плагин-файлы Obsidian не относятся к проектной памяти и создают шум в `git status`.

**Альтернативы:**
- Коммитить `.obsidian/` и синхронизировать персональные настройки между машинами.

## 2026-02-20 — Парсинг tool-calls в chat/completions
**Решение:**
- Для `chat/completions` поддержать оба формата:
  - нативный JSON `tool_calls` / `function_call` из ответа,
  - legacy текстовые блоки `<tool_call>/<function_call>`.
- Преобразовывать их в `ResponseItem::FunctionCall`, чтобы инструменты реально выполнялись в TUI/CLI.

**Причина:**
- Некоторые модели `llmops` возвращают tool-calls текстом, что ранее не исполнялось.
- Нативные `tool_calls` должны исполняться без зависимости от эвристик по тексту.

**Альтернативы:**
- Запретить legacy формат и требовать строго нативные `tool_calls`.
- Парсить только текстовый формат без поддержки JSON `tool_calls`.

## 2026-02-21 — Chat Completions: обязательная передача tools в request
**Решение:**
- В `chat/completions` request обязательно передавать `tools` и `tool_choice=auto` при наличии доступных function-tools.
- Для `aqa-codex-tui` добавить startup prompt токена для non-OpenAI provider, если отсутствуют и `ENV key`, и inline bearer token.

**Причина:**
- Без `tools` в request модель часто отвечает текстовыми инструкциями вместо структурированного `tool_call`, даже если парсер ответа поддерживает `tool_calls`.
- В `llmops` ошибка `400 no healthy deployments` может быть связана с выбранной моделью и не должна трактоваться как проблема токена/клиента.

**Альтернативы:**
- Полагаться только на legacy текстовый fallback-парсинг без отправки `tools` (нестабильно и зависит от конкретной модели).
- Не добавлять startup prompt токена и требовать ручную подготовку env перед каждым запуском.

## 2026-02-22 — Для chat/completions tool-calling нужен real e2e smoke (env-gated), а не только mock
**Решение:**
- В `sb-codex-fork` держать два уровня проверки:
  - `e2e:mock` для стабильной проверки маршрутизации adapter/request mapping;
  - `e2e:llm` (env-gated через `RUN_E2E_LLM=1`) для реальной проверки `llmops` `chat/completions` на структурированный `tool_calls` / `function_call`.
- Реальный e2e smoke должен явно падать, если модель возвращает псевдо-теги (`<tool_call>`, `<function_call>`) текстом вместо структурированного ответа.

**Причина:**
- Моки подтверждают только локальную логику adapter-а и не дают гарантию поведения реального провайдера/модели.
- Для `llmops` поведение зависит от конкретной модели и текущих деплоев; это нужно проверять на живом API.

**Альтернативы:**
- Оставить только `e2e:mock` и проверять живой API вручную через ad-hoc curl/TUI.
- Делать real e2e всегда обязательным в CI (дороже и нестабильнее без секретов/квот).

## 2026-02-22 — Промоут codex-rs chat/completions интеграции в основной поток: `develop -> main`, default branch = `main`
**Решение:**
- После merge feature-веток по `codex-rs`/`chat-completions` в `develop`, выполнить промоут `develop -> main` в основном репозитории `aqa-agent-framework-npm`.
- После промоута вернуть `main` как default branch на GitHub.

**Причина:**
- Задача по замене/интеграции потока на расширенный `codex-rs` доведена до рабочего состояния (код + тесты + merge в `codex-rs/main`), и наработки должны стать базовым состоянием основного репозитория.
- `main` как default branch упрощает навигацию и фиксирует актуальный production baseline после промоута.

**Альтернативы:**
- Оставить `develop` как default branch и откладывать промоут до завершения real `e2e:llm`.
- Делать частичный cherry-pick вместо merge `develop -> main`.

## 2026-02-22 — `llmops` provider integration: support provider-compatible auth and `/models` formats; keep wire API selectable
**Решение:**
- Для built-in `llmops` поддержать совместимый auth/header набор (`Authorization: Bearer`, `X-Copilot-User-Token`, `X-Copilot-User-Agent`).
- Для каталога моделей поддержать OpenAI-compatible shape `{"object":"list","data":[...]}` в дополнение к внутреннему формату.
- Для провайдера `llmops` добавить env override `LLMOPS_WIRE_API=responses|chat-completions` вместо жесткой привязки к одному wire API.
- В TUI prompt токена делать скрытый ввод и прокидывать введенный токен в процессный env, чтобы `/model` мог сразу загрузить remote catalog.

**Причина:**
- Реальный `llmops` API использует смешанную совместимость (OpenAI-like endpoints/JSON + provider-specific headers), и без этого `/model` оставался на fallback.
- Разные модели/деплои `llmops` могут лучше работать либо через `responses`, либо через `chat/completions`, поэтому нужен runtime switch без форка провайдера.
- Токен, введенный только в prompt, должен быть доступен всем runtime paths (включая `ModelsManager` refresh), иначе UI и запросы работают несогласованно.

**Альтернативы:**
- Требовать только `Authorization: Bearer` и один JSON формат `/v1/models` (ломается совместимость с текущим `llmops`).
- Завести отдельные provider names (`llmops-responses`, `llmops-chat`) вместо одного `LLMOPS_WIRE_API` override.
- Требовать ручной `export LLMOPS_TOKEN` перед каждым запуском и не прокидывать prompt token в env.

## 2026-02-22 — Rebrand strategy for codex-rs fork: add `gena` as a branding layer, not a full rename
**Решение:**
- Реализовать ребрендинг форка как минимальный слой поверх upstream-compatible `codex-rs`, а не как тотальный rename всех `codex` идентификаторов.
- Добавить brand aliases (`gena`, `gena-tui`) и dynamic UX-строки по `argv[0]`.
- Для данных/состояния использовать отдельный namespace `~/.gena-codex`, но с мягким fallback на существующий `~/.aqa-codex`.
- Сохранить совместимость со старыми бинарями (`aqa-codex*`) и upstream (`codex*`) на переходный период.

**Причина:**
- Минимальный diff к upstream сильно упрощает регулярный sync новых фич из оригинального `codex-rs` и снижает конфликтность merge.
- Полный rename бренда по всему коду и документации создает большой шум и дорого сопровождается при каждом upstream merge.
- Отдельный home-dir namespace (`~/.gena-codex`) изолирует state нового бренда, но fallback на `~/.aqa-codex` сохраняет бесшовный переход для текущих пользователей.

**Альтернативы:**
- Полный rename всех `codex` строк/типов/путей в форке (максимальная конфликтность с upstream).
- Только symlink/alias без brand-aware UX и без отдельного state namespace (пользовательский опыт непоследователен).
- Новый брендовый home-dir `~/.gena` (короче, но более общий namespace с риском конфликтов и меньшей самодокументируемостью).

## 2026-02-23 — Brand-specific project config namespace for `gena`/`aqa` forks in codex-rs
**Решение:**
- Для fork-брендов в `codex-rs` выбирать project config directory по basename `CODEX_HOME`:
  - `~/.gena-codex` -> искать `./.gena-codex/config.toml`
  - `~/.aqa-codex` -> искать `./.aqa-codex/config.toml`
  - fallback -> `./.codex/config.toml`
- Логику реализовать в `core/src/config_loader/mod.rs` (project layer discovery), а не обходить это локальными workaround'ами в TUI.

**Причина:**
- `gena-tui` корректно сохранял выбор модели в `~/.gena-codex/config.toml`, но при запуске из `/root` подхватывал project config из `/root/.codex/config.toml`, который перетирал модель на `gpt-5.3-codex`.
- Это конфликт namespace'ов между upstream-брендом (`codex`) и fork-брендом (`gena`) на уровне project config layers.
- Исправление на уровне config loader сохраняет upstream-friendly архитектуру и исключает повторение проблемы для `aqa-codex*`.

**Альтернативы:**
- Удалять/переименовывать `/root/.codex/config.toml` вручную (неустойчиво и ломает сценарии upstream `codex`).
- Запускать `gena-tui` только вне директорий с `.codex/` (не решает архитектурно).
- Отключить project config layers полностью для `gena` (теряется полезная возможность брендового project-level конфигурирования).

## 2026-02-23 — Для aarch64 Linux на этой машине использовать `gcc + mold` как linker driver для Rust (вместо `clang + mold`)
**Решение:**
- В `codex-rs/.cargo/config.toml` добавить platform-specific linker config для `aarch64` Linux:
  - `linker = "gcc"`
  - `-fuse-ld=mold`
  - `-Wl,--no-keep-memory`
- Не использовать `clang + mold` в этом окружении, оставить `gcc` как driver для `mold`.

**Причина:**
- `clang + mold` в данном proot/Ubuntu-on-Termux окружении падал на ранней линковке с ошибкой `mold: fatal: library not found: gcc_s`.
- `gcc + mold` корректно разрешает системные runtime/lib paths (`libgcc_s`) и позволяет выполнять сборку.
- Основной прирост дает сам `mold`; выбор `gcc` vs `clang` как driver обычно влияет меньше, чем стабильность линковки.

**Подтверждение:**
- `mold` установлен через `apt`.
- `cargo build -p codex-tui --bin gena-tui -j1` с конфигом `gcc + mold` проходит.
- Warm rebuild (`gena-tui`) после прогрева артефактов: ~18s (`real 0m18.020s`).

**Альтернативы:**
- `clang + mold` с ручной настройкой toolchain/runtime paths (`gcc_s`) — потенциально исправимо, но невыгодно по усилиям при рабочем `gcc + mold`.
- `gcc + lld` / `clang + lld` — рабочие варианты, но без явного преимущества над текущей стабильной конфигурацией `gcc + mold`.

## 2026-02-23 — Обновление `codex-rs` форка с `upstream/main` выполнять через отдельную sync-ветку и compile-driven two-pass процесс
**Решение:**
- Для апдейта long-lived форка `codex-rs` использовать отдельную ветку вида `chore/upstream-sync-YYYY-MM-DD`.
- Выполнять sync в два этапа:
  1. `merge upstream/main` + first-pass conflict resolution для кастомных зон форка;
  2. compile-driven second pass (`cargo check`) с адаптацией API drift до compile-green baseline.
- Только после compile-green и runtime smoke рассматривать merge sync-ветки в `main`.

**Причина:**
- Форк имеет существенный слой кастомизаций (`llmops`, `chat/completions`, `gena` branding, `gena-tui` UX/persistence), поэтому прямой merge апстрима дает ожидаемые конфликты и API-сдвиги в `core`/`tui`.
- Реальный прогон показал, что механизм воспроизводим и управляем, но требует отдельного интеграционного окна; это не "быстрый" апдейт.
- Разделение на merge-pass и compile-pass снижает риск хаотичных правок и упрощает triage ошибок.

**Подтверждение:**
- Ветка `chore/upstream-sync-2026-02-23`.
- First-pass merge commit: `0dfd6fd4b`.
- Compile-green baseline fix commit: `bf4bb136f`.
- `cargo check -p codex-core -j1` PASS; `cargo check -p codex-tui -j1` PASS.

**Ограничение/операционный риск:**
- Push sync-ветки может блокироваться GitHub token scope (`workflow`), если merge затрагивает `.github/workflows/*`; для публикации нужен token с `workflow` scope или другой способ push.

**Альтернативы:**
- Пытаться делать регулярный `rebase` форка поверх `upstream/main` (выше стоимость конфликтов при long-lived customizations).
- Мержить `upstream/main` напрямую в `main` без sync-ветки (хуже управляемость, выше риск сломать рабочую ветку).

## 2026-02-24 — При ограниченном GitHub token (`workflow` scope отсутствует) публиковать upstream-sync ветку через workflow-free commit
**Решение:**
- Если `git push` sync-ветки блокируется GitHub из-за изменений в `.github/workflows/*`, публиковать отдельный workflow-free вариант sync-коммита (исключить `.github/workflows/*` из публикуемого коммита).
- Полный baseline с workflow-изменениями сохранять локально отдельной веткой для последующей публикации при наличии токена с `workflow` scope.

**Причина:**
- Нужно сохранить и поделиться результатом upstream-sync (compile-green baseline) без задержки на переоформление токена.
- GitHub блокирует обновление workflow-файлов для OAuth app token без `workflow` scope, даже если остальная часть ветки валидна.

**Применение (в этой сессии):**
- Remote ветка `origin/chore/upstream-sync-2026-02-23` опубликована workflow-free publishable commit.
- Локально сохранен полный вариант в `chore/upstream-sync-2026-02-23-fullhistory-local`.

**Альтернативы:**
- Получить токен с `workflow` scope и пушнуть full-history ветку как есть.
- Оставить sync-ветку только локально (хуже для продолжения работы с других машин/сессий).

## 2026-02-24 — Версионировать upstream-базу форка по релизам Codex Rust с явным префиксом `codex-rust-`
**Решение:**
- Для фиксации upstream-базы форка использовать именование вида `codex-rust-vX.Y.Z`.
- В `codex-rs` хранить anchor metadata в `UPSTREAM_BASE.md`, включая:
  - `upstream_release` (internal alias, например `codex-rust-v0.104.0`)
  - `upstream_tag_ref` (`rust-v0.104.0`)
  - `upstream_commit` (SHA тега)
  - дату тега и remote URL.
- Release-based sync ветки именовать по тому же шаблону:
  - `chore/upstream-sync-codex-rust-vX.Y.Z`.

**Причина:**
- Релизные теги апстрима — более стабильная и понятная база синхронизации, чем `upstream/main`.
- Префикс `codex-rust-` делает название самодокументируемым (продукт + линия реализации + версия).
- Упрощает коммуникацию и changelog форка: "наша база = codex-rust-v0.104.0".

**Альтернативы:**
- Использовать только `rust-vX.Y.Z` без внутреннего alias (менее наглядно в контексте форка).
- Продолжать ориентироваться на `upstream/main` как основную sync-цель (выше риск нестабильных интеграций).

## 2026-02-24 — Release-based sync от `codex-rust-v0.104.0` оказался значительно дешевле по конфликтам/compile-fixes, чем sync от `upstream/main`
**Решение:**
- Для production-oriented sync форка использовать release-based ветки от upstream тегов (`codex-rust-vX.Y.Z`) как основной путь интеграции.
- Sync от `upstream/main` оставить для разведки/ранней оценки API drift и механики merge.

**Причина:**
- Реальный merge `rust-v0.104.0` дал всего 3 конфликтных файла на first-pass.
- Compile-driven pass до baseline (`codex-core` + `codex-tui`) потребовал небольшой пакет точечных фиксов (в основном вызовы `ModelsManager` с `&Config`).
- Для `upstream/main` ранее объем конфликтов и API drift был существенно выше.

**Подтверждение:**
- Ветка `chore/upstream-sync-codex-rust-v0.104.0`:
  - `f69912225` — first-pass merge conflict resolution
  - `bfd532c8b` — compile-green fixes
- `cargo check -p codex-core -j1` PASS
- `cargo check -p codex-tui -j1` PASS
- Remote publishable workflow-free commit:
  - `eccd113b6` (из-за отсутствия `workflow` scope у токена)

**Альтернативы:**
- Продолжать синхронизировать production baseline напрямую от `upstream/main` (дороже и менее предсказуемо).

## 2026-02-25 — Для `llmops + responses` применять provider-level compatibility guards (omit unsupported params/tools)
**Решение:**
- Для built-in `llmops` в режиме `responses` добавлять compatibility-guards на уровне клиента до отправки запроса:
  - unsupported/нежелательные параметры делать `omit` (не отправлять ключ в JSON), а не просто выставлять `false`;
  - provider-problematic tools (например `web_search` в данном маршруте) вырезать из toolset до отправки модели.
- Переключение wire API (`responses` / `chat-completions`) оставить управляемым через config/profile/env.

**Причина:**
- Реальная проверка показала, что `litellm/xAI` route может падать не только на значениях параметров, но и на самом факте присутствия ключа (`parallel_tool_calls`), даже если значение `false`.
- Аналогично route вернул `410 Gone` на live search в `responses`, поэтому generic provider-level guard по `web_search` для `llmops + responses` снижает хрупкость маршрута.
- Это не ломает `chat/completions` path и позволяет сохранить единый UX-переключатель wire API.

**Подтверждение:**
- `feature/llmops-wire-api-switch` -> `codex-rs/main`
- Commit: `536d7ef65`
- Merge: `a3d5ca2c7`
- Runtime: исчезли blockers `UnsupportedParamsError(['parallel_tool_calls'])` и `xAI live search deprecated (410)` в `llmops + responses`.

## 2026-03-09 — После thinning `codex-cli` следующий приоритет архитектуры `gena` смещается в общий bootstrap/runtime path `exec` и `tui`
**Решение:**
- После того как `codex-cli` стал заметно тоньше, дальнейший приоритет отдавать не endless local cleanup в `cli`, а выносу общего bootstrap/config/startup knowledge из `codex-exec` и `codex-tui` в `gena-runtime`.
- Считать целевыми кандидатами на вынос:
  - config build path,
  - `fallback_cwd`-вариант config build,
  - execpolicy validation,
  - residency/login post-config checks,
  - затем OSS/provider bootstrap path.

**Причина:**
- Эти зоны более upstream-sensitive, чем оставшийся dispatch glue в `cli`.
- Их централизация сильнее снижает стоимость будущих sync с Codex upstream.
- Такой путь сохраняет additive migration и не требует big-bang rewrite repo.

**Подтверждение:**
- Реализовано в `chore/update-arch` и опубликовано:
  - `917daec5b` — `refactor(gena): centralize runtime bootstrap checks`
- После этого `codex-exec` и `codex-tui` используют общий `gena-runtime` path для:
  - runtime config build,
  - `fallback_cwd` config build,
  - execpolicy validation,
  - residency/login startup checks.

**Альтернативы:**
- Продолжать локально полировать `cli/src/main.rs` helper'ами (меньший выигрыш для upstream sync).
- Переходить к `plugin-api` раньше стабилизации bootstrap/runtime boundaries (слишком рано и даёт меньший практический эффект).

## 2026-03-09 — OSS provider planning/readiness тоже нужно держать в `gena-runtime`, а не в entrypoints
**Решение:**
- Политику bootstrap вокруг OSS provider централизовать в `gena-runtime`, включая:
  - headless path с явной ошибкой при отсутствии default provider,
  - interactive path с decision `resolved vs needs selection`,
  - readiness check уже после загрузки итогового config.

**Причина:**
- Это такой же upstream-sensitive bootstrap knowledge, как config build и execpolicy checks.
- Если оставить этот код размазанным в `codex-exec` и `codex-tui`, будущие sync снова будут чаще конфликтовать именно в entrypoints.
- Вынос в runtime остаётся additive и не требует тащить UI-зависимости внутрь `gena-runtime`.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `d858a5362` — `refactor(gena): unify oss bootstrap runtime flow`
- После этого `codex-exec` и `codex-tui` используют общий `gena-runtime` path для:
  - OSS provider planning,
  - headless provider resolution policy,
  - interactive provider-selection planning,
  - OSS provider readiness checks.

**Альтернативы:**
- Оставить OSS bootstrap policy внутри entrypoints (больше будущий diff к upstream).
- Перенести это позже после `plugin-api` (неоптимальный порядок работ).

## 2026-03-09 — В `codex-tui` оставлять только UI/IO часть provider bootstrap, а non-UI policy уносить в `gena-runtime`
**Решение:**
- Для provider token bootstrap в `tui` оставлять только интерактивный prompt/read flow.
- Все не-UI части держать в `gena-runtime`, включая:
  - проверку `requires_openai_auth`,
  - проверку env/inline token,
  - sidecar load/persist policy,
  - state `ready vs needs prompt`,
  - trust-screen decision policy.

## 2026-03-12 — Первый deeper plugin integration step делать через runtime-owned command dispatch plan, а не через loader/discovery
**Решение:**
- Следующий шаг после command gate делать через command dispatch plan:
  - `CommandPlugin` регистрирует dispatch descriptor
  - runtime строит `RuntimeCommandDispatchPlan`
  - CLI использует этот план как source of truth для execution path

**Причина:**
- Это даёт следующий уровень plugin integration без premature loader/discovery.
- Такой шаг уже выходит за рамки introspection/gating, но ещё не требует dynamic execution framework.
- Он лучше соответствует DoD, чем просто ещё один built-in plugin descriptor.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `441b095ef` — `feat(gena): add plugin command dispatch plan`

**Альтернативы:**
- Сразу делать loader/discovery (слишком широко).
- Добавлять только второй built-in consumer без нового dispatch contract (меньший ROI).
- Пытаться сразу сделать полностью dynamic plugin execution (слишком дорогой шаг для текущей фазы).

## 2026-03-12 — После `voice` runtime step цикл дешёвых production seam refactor'ов можно считать в основном закрытым
**Решение:**
- После шага `ec6fac82c` считать phase дешёвых production seam extraction в основном завершённой.
- Не продолжать механически искать ещё один helper-step только ради движения.
- Следующий шаг выбирать уже как новый крупный архитектурный этап.

**Причина:**
- Почти все дешёвые и средние по цене production seams уже вытащены в Gena-owned слои.
- Оставшиеся маленькие хвосты дают заметно меньший ROI для upstream sync.
- Дальнейшие meaningful изменения уже лежат либо в deeper plugin integration, либо в более дорогих runtime/UI orchestration этапах.

**Подтверждение:**
- Checkpoint зафиксирован после опубликованного шага:
  - `ec6fac82c` — `refactor(gena): centralize voice auth context`

**Альтернативы:**
- Продолжать делать low-ROI residual cleanup шаги один за другим.
- Форсировать следующий крупный этап без явной переоценки его цены и цели.

## 2026-03-12 — `voice` auth/bootstrap path тоже должен жить в `gena-runtime`, если он не содержит UI
**Решение:**
- Product home resolution, persisted auth read, config load и ChatGPT base URL normalization для `voice` transcription path держать в `gena-runtime`, а не в `tui/voice`.

**Причина:**
- Это bootstrap/runtime policy glue, а не UI-логика.
- Оставление его в `tui/voice` держит ещё один production auth path вне Gena-owned слоя.
- Такой вынос дешёвый, compile-checkable и не требует тащить в runtime audio IO или request orchestration.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `ec6fac82c` — `refactor(gena): centralize voice auth context`

**Альтернативы:**
- Оставить `resolve_auth()` локально в `tui/voice` (хуже для crate boundary и upstream sync).
- Пытаться сразу унести в runtime всю transcription request orchestration (слишком широкий шаг).

## 2026-03-12 — Первый реальный plugin execution path должен идти через runtime-owned command resolution, а не через loader/discovery
**Решение:**
- Первый production-facing execution step для plugin phase делать не через полноценный plugin loader, а через очень узкий runtime-owned command registry seam:
  - registry должен знать ownership `plugin_id -> command_name`
  - runtime должен уметь resolve/require command plugin по имени
  - entrypoint command path может использовать этот runtime seam как execution gate

**Причина:**
- Это даёт первый реальный execution path поверх plugin registry без premature loader/discovery.
- Такой шаг остаётся compile-checkable и не требует выносить clap parsing или полноценную dynamic dispatch систему.
- Он лучше отражает target architecture, чем просто ещё один debug/introspection helper.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `9871dff50` — `feat(gena): add runtime-owned plugin command path`

**Альтернативы:**
- Сразу вводить plugin loader/discovery (слишком рано и слишком широко).
- Добавить ещё один purely introspection-only consumer без production path (меньший ROI).
- Пытаться сразу сделать полностью dynamic command dispatch через plugin registry (слишком дорогой шаг для текущей стадии).

## 2026-03-12 — Plugin registry contract не должен дублировать metadata и не должен тянуть обратную зависимость `gena-plugins-core -> gena-runtime`
**Решение:**
- `PluginRegistry` должен хранить plugin metadata как список уникальных plugins, а capability names как отдельные коллекции.
- `gena-plugins-core` не должен зависеть от `gena-runtime` только ради пустой инициализации registry.
- CLI usage path для built-in plugins должен идти через runtime seam, а не через локальный обходной builder в `gena-plugins-core`.

**Причина:**
- Один plugin может реализовывать несколько capability category; в таком случае дубли metadata искажают сам контракт registry.
- Зависимость `gena-plugins-core -> gena-runtime` нарушает желаемое направление зависимостей и делает built-in plugins слишком связанными с runtime internals.
- Если первый usage path обходит runtime seam, то позже появятся два источника истины: runtime registry и локально собранный CLI registry.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `49a237dc8` — `refactor(gena): tighten plugin registry boundaries`

**Альтернативы:**
- Оставить duplicate metadata как “норму” для prework (плохо масштабируется на multi-capability plugins).
- Сохранить `gena-plugins-core -> gena-runtime` ради простоты тестов и builder-функции (портит crate boundaries).
- Оставить `debug plugins` на прямом вызове `build_core_plugin_registry()` (создаёт обход runtime integration seam).

**Причина:**
- Это не UI-логика, а bootstrap/runtime policy.
- Если оставить её в `tui`, future diff к upstream по-прежнему будет концентрироваться в entrypoint crate.
- Такой вынос сохраняет чистую границу: runtime решает policy/state, `tui` занимается user interaction.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `dffd05cce` — `refactor(gena): move tui bootstrap policy into runtime`
- После этого `codex-tui` держит в provider bootstrap в основном:
  - prompt rendering,
  - secret reading,
  - user-facing error/reporting.

**Альтернативы:**
- Оставить sidecar/env/trust policy в `tui` (хуже для upstream sync).
- Пытаться унести в runtime и сам prompt IO (сломает clean boundary между policy и UI).

## 2026-03-09 — Onboarding/login screen decisions в `tui` тоже нужно держать в `gena-runtime`, если это pure policy
**Решение:**
- Decision helpers вида:
  - показывать ли login screen,
  - показывать ли onboarding вообще,
держать в `gena-runtime`, если они зависят только от `Config` и runtime state, а не от UI.

**Причина:**
- Это такая же bootstrap policy, как trust-screen policy.
- Оставление её в `tui` не даёт архитектурной пользы и сохраняет лишний future diff в UI crate.
- При этом сам onboarding flow и screen rendering должны оставаться в `tui`.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `fc62c0e62` — `refactor(gena): move tui onboarding policy into runtime`

**Альтернативы:**
- Оставить onboarding/login decision helpers в `tui` (хуже crate boundary).
- Пытаться унести в runtime уже сам onboarding flow (слишком далеко, это UI territory).

## 2026-03-09 — После переноса auth status в `gena-runtime` цикл micro-thinning `codex-tui` можно считать закрытым
**Решение:**
- После переноса:
  - trust-screen policy,
  - onboarding/login decision policy,
  - provider token non-UI policy,
  - auth status resolution,
дальше не продолжать механически вытаскивать мелкие helper'ы из `codex-tui`.
- Следующий этап выбирать уже как новый architectural boundary с лучшей отдачей.

**Причина:**
- Дальнейшая отдача от micro-thinning `tui` становится низкой.
- Растёт риск начать размывать UI/runtime boundary ради косметических выносов.
- Текущая граница уже хорошая: runtime держит policy/state, `tui` держит interaction/UI orchestration.

**Подтверждение:**
- Финальный micro-step этого цикла:
  - `c557d209c` — `refactor(gena): move tui auth status into runtime`

**Альтернативы:**
- Продолжать искать ещё один helper в `tui` (низкий ROI).
- Остановиться и перейти к следующему architectural step (предпочтительно).

## 2026-03-09 — Следующий high-ROI boundary после `tui` thinning: auth/cloud bootstrap factories
**Решение:**
- После закрытия цикла micro-thinning `codex-tui` следующим boundary выбрать вынос auth/cloud bootstrap factories в `gena-runtime`, а именно:
  - построение `AuthManager`
  - построение `CloudRequirementsLoader`
- Использовать эти helper'ы и внутри runtime bootstrap, и в `codex-exec` / `codex-tui`.

**Причина:**
- Это повторяющийся integration glue, который всё ещё оставался размазан по нескольким слоям.
- Он ближе к runtime/bootstrap boundary, чем к UI или entrypoint-specific logic.
- Отдача от такого выноса выше, чем от дальнейшего поиска мелких helper'ов в `tui`.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `46b10ed5e` — `refactor(gena): centralize auth and cloud bootstrap`

**Альтернативы:**
- Сразу идти в `plugin-api` (слишком рано).
- Продолжать micro-thinning `tui` (низкий ROI).
## 2026-03-13 — Первый real provider execution path должен идти через `llmops` bootstrap, а не через synthetic debug consumer
**Решение:**
- Для следующего шага в `Plugin Platform Expansion` первым real `provider execution` path выбран built-in `llmops` provider.
- Execution wiring проводить через уже существующие production bootstrap paths в:
  - `codex-exec`
  - `codex-tui`
- Не открывать на этом шаге:
  - dynamic provider execution framework
  - provider selection orchestration через plugins
  - synthetic debug-only consumer

**Причина:**
- `provider` axis уже существовал как descriptor/introspection layer, но без real execution path.
- Самый дешёвый и честный production consumer здесь — built-in `llmops`, потому что он уже является частью реального provider/bootstrap landscape.
- Переводить сразу весь provider/model-selection flow на plugins было бы слишком дорогим скачком и открыло бы новый широкий scope.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `f76f1b5ce` — `feat(gena): add llmops provider execution`

**Альтернативы:**
- Делать synthetic debug-only provider consumer (слишком слабый architectural payoff).
- Сразу переводить весь provider bootstrap/model-selection на plugin execution (слишком большой фронт).
- Идти сперва в dynamic loader (преждевременно).
## 2026-03-13 — После real provider execution следующим шагом нужны built-in lifecycle callbacks, а не сразу dynamic loader
**Решение:**
- После шага `f76f1b5ce` следующим execution/lifecycle frontier выбрать built-in runtime lifecycle callbacks.
- Реализовать callbacks только внутри `gena-runtime` вокруг уже существующих real execution paths:
  - `mcp`
  - `llmops`
  - `apply_patch`
  - `review`
- Не открывать на этом шаге:
  - plugin-provided hooks
  - dynamic loader
  - новые execution axes

**Причина:**
- Execution stack уже имел:
  - static discovery
  - snapshot
  - lifecycle manager
  - unified planning
  - prepared execution
  - lifecycle trace
- Следующий правильный шаг — сделать lifecycle execution-aware, а не сразу переходить к dynamic loading.
- Это даёт более зрелый общий каркас до открытия ещё более дорогих фронтов.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `c36098d4e` — `feat(gena): add builtin plugin lifecycle callbacks`

**Альтернативы:**
- Сразу идти в dynamic loader (преждевременно и дороже).
- Сразу добавлять plugin-provided hooks (слишком широкий следующий фронт).
- Открывать ещё одну capability execution axis (архитектурно слабее, чем углубить общий lifecycle слой).
## 2026-03-13 — После built-in lifecycle callbacks следующий правильный шаг — plugin-provided lifecycle hooks, а не dynamic loader
**Решение:**
- После `c36098d4e` следующим execution/lifecycle frontier выбрать plugin-provided lifecycle hooks.
- Hook contract проводить через `gena-plugin-api` и discovered `PluginRegistry`, а не через runtime-hardcoded callback map.
- На этом шаге не открывать:
  - dynamic loader
  - lifecycle callbacks как executable trait objects
  - новые execution axes

**Причина:**
- Built-in callback layer уже доказал полезность lifecycle stage/callback trace.
- Следующий правильный архитектурный шаг — сделать callbacks plugin-owned, а не добавлять ещё больше hardcoded runtime knowledge.
- Это углубляет plugin platform без скачка сразу в dynamic loading.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `41707e244` — `feat(gena): add plugin-provided lifecycle hooks`

**Альтернативы:**
- Сразу переходить к dynamic loader (слишком дорогой и преждевременный следующий фронт).
- Оставить lifecycle callbacks hardcoded в runtime (хуже для plugin-driven architecture).
- Добавлять новые execution axes до углубления lifecycle ownership (архитектурно слабее).
## 2026-03-13 — После plugin-provided lifecycle hooks следующий правильный шаг — dynamic loader как runtime-owned registrar bridge
**Решение:**
- После `41707e244` следующим шагом открыть `dynamic loader`, но в минимальном виде:
  - без полного plugin execution framework rewrite
  - без new config schema
  - без ecosystem/publishing layer
- Dynamic loader реализовать как runtime-owned bridge поверх существующего static discovery:
  - canonical exported registrar symbol
  - env-driven library path list
  - loading через `libloading`

**Причина:**
- Plugin platform foundation уже имела:
  - capability axes
  - static discovery
  - unified execution planning
  - lifecycle manager
  - prepared execution
  - lifecycle trace
  - plugin-provided lifecycle hooks
- Следующий естественный foundation step — dynamic loading bridge, а не ещё один hardcoded/runtime-local frontier.
- Это даёт честный путь к external plugins, не открывая сразу всю ecosystem phase.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `80e68a55c` — `feat(gena): add dynamic plugin loader`

**Альтернативы:**
- Остановиться на static discovery + lifecycle hooks (сильный, но ещё не полный plugin platform checkpoint).
- Сразу открывать full external ecosystem phase (слишком широкий скачок).
- Тащить dynamic loading в `cli/tui/exec` без runtime-owned bridge (хуже boundary).
## 2026-03-14 — После ranked + fuzzy registry search следующий правильный шаг — richer registry metadata contract, а не backend/service rewrite
**Решение:**
- После `18823ebd0` следующим шагом углубить remote registry/index contract через richer metadata:
  - `publisher`
  - `license`
  - `repository_url`
- Делать это внутри существующего TOML index format и current runtime parser.
- Не открывать на этом шаге:
  - новый registry backend/service
  - auth
  - pagination
  - remote API redesign

**Причина:**
- Registry story уже включала:
  - inspect
  - ranked + fuzzy search
  - registry-backed install / replace / uninstall
- Следующий дешёвый, но полезный шаг — сделать service-facing metadata contract богаче, не открывая сразу отдельный backend frontier.
- Это усиливает packaging/distribution layer и улучшает operator-visible output без лишнего network scope.

**Подтверждение:**
- Реализовано и опубликовано в `chore/update-arch`:
  - `54fb940fb` — `feat(gena): enrich plugin registry metadata`

**Альтернативы:**
- Сразу открывать richer registry service/backend contract.
- Идти в pagination/search backend semantics.
- Продолжать только installer UX без усиления registry metadata.
## 2026-03-14 — После richer registry metadata следующим шагом должен быть explicit discovery/service contract в current index, а не новый backend слой
**Решение:**
- Не открывать новый registry backend/service protocol.
- Сначала расширить existing TOML index contract полями:
  - `summary`
  - `capabilities`
  - `tags`
- Подключить эти поля к current runtime search/ranking layer.

**Причина:**
- После `54fb940fb` registry уже имел publisher/license/repository metadata, но всё ещё был слабым как discovery contract.
- Следующий cheapest useful step — сделать current index format богаче семантически:
  - identity
  - metadata
  - summary
  - capabilities
  - tags
- Это усиливает search/discovery UX без backend rewrite.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `986cdb28f` — `feat(gena): extend plugin registry contract`

**Альтернативы:**
- Сразу открывать richer registry backend/service contract.
- Идти в pagination/search protocol versioning.
- Продолжать только package/install management без усиления discovery surface.
## 2026-03-14 — После explicit discovery fields следующим шагом должен быть top-level index manifest, а не backend/service rewrite
**Решение:**
- После `986cdb28f` расширить current TOML registry/index contract top-level `[index]` manifest:
  - `format_version`
  - `registry_id`
  - `display_name`
- Добавить runtime helper для explicit manifest inspection.
- Не открывать на этом шаге:
  - новый backend/service layer
  - pagination
  - auth
  - index protocol negotiation beyond one supported version

**Причина:**
- После `summary/capabilities/tags` package entries стали богаче, но сам index format всё ещё оставался implicit.
- Следующий cheapest useful step beyond current package metadata — явная top-level manifest boundary для самого index.
- Это даёт protocol/version/source contract без backend rewrite.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `a9920d32a` — `feat(gena): add plugin index manifest`

**Альтернативы:**
- Сразу открывать richer registry backend/service contract.
- Сразу идти в pagination or auth semantics.
- Оставить index format implicit на уровне only `[[packages]]`.
## 2026-03-14 — После top-level index manifest следующим шагом должны быть service-aware index capabilities, а не backend rewrite
**Решение:**
- После `a9920d32a` расширить `[index]` manifest полем:
  - `capabilities`
- Описывать через него supported registry actions текущего index contract.
- Не открывать на этом шаге:
  - backend/service layer
  - auth
  - pagination
  - remote protocol negotiation

**Причина:**
- Top-level manifest уже дал protocol boundary, но сам contract всё ещё был слабым как service description.
- Следующий cheapest useful step — описать, какие registry actions current index реально поддерживает.
- Это усиливает service-awareness без backend rewrite.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `acfb22acf` — `feat(gena): add index service capabilities`

**Альтернативы:**
- Сразу открывать richer backend/service contract.
- Идти в auth/pagination semantics.
- Оставить `[index]` manifest без service capability surface.
## 2026-03-14 — После service-aware index capabilities следующим шагом должны быть optional service endpoints, а не backend rewrite
**Решение:**
- После `acfb22acf` расширить registry contract optional section:
  - `[service]`
    - `search_url`
    - `package_url_template`
- На этом шаге только описывать и валидировать endpoints.
- Не переключать текущие install/search/manage flows на эти endpoints автоматически.

**Причина:**
- `[index].capabilities` уже сделал contract service-aware, но сам service surface всё ещё был только словарём возможностей.
- Следующий cheapest useful step — явно описать endpoint-level service contract без backend rewrite.
- Это даёт first bridge к future service-backed registry, не ломая current static TOML flows.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `353608950` — `feat(gena): add registry service endpoints`

**Альтернативы:**
- Сразу переключать current flows на service endpoints.
- Сразу открывать auth/pagination/service backend.
- Оставить service contract на уровне only `[index].capabilities`.
## 2026-03-14 — После service endpoints следующим шагом должен быть один service-backed path с fallback, а не полный backend switch
**Решение:**
- После `353608950` сделать first service-backed registry path через:
  - service-backed search
- Если `[service].search_url` объявлен:
  - runtime идёт в него с `q=<query>`
- Если endpoint не объявлен:
  - runtime сохраняет existing static index search path
- Не открывать на этом шаге:
  - auth
  - pagination
  - package detail endpoint usage
  - mandatory service mode

**Причина:**
- Endpoint-level contract уже появился, но без реального consumer это оставалось только metadata.
- Следующий cheapest useful step — один реальный service-backed path, но с fallback на current static flow.
- Search для этого лучше всего подходит: минимальный риск и хороший operator payoff.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `5a0668f25` — `feat(gena): add service-backed registry search`

**Альтернативы:**
- Сразу переключать install/manage на service endpoints.
- Сразу открывать auth/pagination/service backend.
- Оставить service endpoints purely descriptive.
## 2026-03-14 — После service-backed search следующим шагом должен быть service-backed package resolution с fallback, а не полный service switch
**Решение:**
- После `5a0668f25` добавить service-backed package resolution через:
  - `[service].package_url_template`
- Runtime должен:
  - строить package URL через `{package_id}`
  - читать `[package]` TOML document
  - валидировать resolved `package_id`
- Если endpoint не объявлен:
  - сохранять current static index lookup fallback
- Не открывать на этом шаге:
  - auth
  - pagination
  - mandatory service mode
  - full remote service rewrite

**Причина:**
- Search уже стал первым реальным service-backed path.
- Следующий самый естественный и дешёвый consumer — package resolution for install/replace/uninstall.
- Это даёт второй service-backed path, не ломая current static flows.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `ebdacc752` — `feat(gena): add service-backed package resolution`

**Альтернативы:**
- Сразу переключать весь registry management на mandatory service mode.
- Сразу открывать auth/pagination.
- Оставить service layer только на search.
## 2026-03-14 — После service-backed package resolution следующим шагом должен быть package detail inspection, а не auth/paging сразу
**Решение:**
- После `ebdacc752` добавить один operator-facing path:
  - `inspect-plugin-from-index <INDEX_URL> <PACKAGE_ID>`
- Реализовать его как thin helper поверх existing resolver.
- Не открывать на этом шаге:
  - auth
  - pagination
  - service-only mode
  - package mutation beyond current install/replace/uninstall loop

**Причина:**
- Search и package resolution уже стали service-backed.
- Следующий cheapest useful step — дать оператору explicit package detail inspect path перед action flows.
- Это усиливает service-backed registry UX без открытия ещё одного transport/backend scope.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `115ff1e57` — `feat(gena): add package detail inspection`

**Альтернативы:**
- Сразу идти в auth.
- Сразу открывать pagination.
- Оставить detail inspection implicit inside install/replace/uninstall only.
## 2026-03-14 — После package detail inspection следующим шагом должен быть optional service auth, а не pagination
**Решение:**
- После `115ff1e57` расширить optional `[service]` contract полем:
  - `auth_env`
- Service-backed search и package resolution должны использовать bearer auth из указанной env var.
- Static index fallback path не трогать.
- Не открывать на этом шаге:
  - pagination
  - mandatory auth mode
  - credential storage
  - token refresh

**Причина:**
- Search, package resolution и package detail уже стали service-backed.
- Следующий cheapest useful step в auth/paging frontier — optional bearer auth for existing service endpoints.
- Это позволяет service registry быть реально protected, не ломая existing fallback model.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `fa260a179` — `feat(gena): add registry service auth`

**Альтернативы:**
- Сразу идти в pagination.
- Сразу делать mandatory auth everywhere.
- Оставить service-backed endpoints auth-blind.
## 2026-03-14 — После optional service auth следующим шагом должен быть search paging, а не full pagination model
**Решение:**
- После `fa260a179` расширить optional `[service]` contract полями:
  - `page_param`
  - `page_size_param`
- Добавить page-aware service-backed search request construction.
- CLI должен получить:
  - `--page`
  - `--page-size`
- Не открывать на этом шаге:
  - next-page tokens
  - service response cursors
  - package list pagination loop
  - mandatory paging model

**Причина:**
- Auth уже сделал service-backed endpoints production-realistic.
- Следующий cheapest useful paging step — дать request-side page controls для search, не открывая ещё cursor protocol.
- Это усиливает service-backed search без большого backend rewrite.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `5f62b31b1` — `feat(gena): add registry search paging`

**Альтернативы:**
- Сразу идти в cursor-based pagination.
- Сразу вводить paged service response metadata.
- Оставить paging только как идея в manifest.
## 2026-03-14 — После request-side paging следующим шагом должен быть paged response metadata, а не cursor protocol сразу
**Решение:**
- После `5f62b31b1` добавить optional response metadata contract:
  - `[response]`
    - `page`
    - `page_size`
    - `total`
    - `next_page`
- Runtime search-with-paging должен возвращать structured page object.
- Старый search API сохранить как совместимый wrapper.
- Не открывать на этом шаге:
  - cursor tokens
  - next-page fetching loop
  - package list iteration framework

**Причина:**
- Request-side paging уже делал service search page-aware, но без response metadata оператор не видел page context.
- Следующий cheapest useful step — ответная paging metadata, не перепрыгивая сразу к cursor protocol.
- Это готовит следующий frontier без ломки current operator UX.

**Подтверждение:**
- Реализовано и compile-green проверено в `chore/update-arch`:
  - `18c00538f` — `feat(gena): add paged search responses`

**Альтернативы:**
- Сразу идти в cursor protocol.
- Сразу делать full next-page loop.
- Оставить paging request-only.

## 2026-05-02 — Gena Chat Completions action preamble без tool call не считается финальным ответом
**Решение:**
- Для LLMOps Chat Completions добавить один retry, если assistant content выглядит как preamble к действию и заканчивается `:`, но `tool_calls` пустой.
- Retry добавляет continuation nudge: вызвать подходящий tool или дать финальный ответ, если tool не нужен.
- Gate должен включать behavioral test через `ModelClientSession::stream`, а не только helper predicate.

**Причина:**
- Реальный `gena-debug` turn завершался после фразы `Посмотрим общий список крупных файлов и папок в корне:` без следующего command execution.
- Это не transport panic и не TUI render bug; это неправильная трактовка неполного Chat Completions ответа как финального.

**Подтверждение:**
- Реализовано в `gena-rs-project`:
  - `bd164e1e1` — `fix(gena): retry chat action preamble without tool call`
- Проверено focused-тестами:
  - `chat_completion_retries_action_preamble_without_tool_call`
  - `chat_completion_detects_action_preamble_without_tool_call`

**Альтернативы:**
- Всегда форсировать tool_choice при наличии tools.
- Считать любой assistant content с `:` незавершённым.
- Игнорировать provider quirk и требовать ручного retry от пользователя.

## 2026-05-04 — Assistant text перед tool call в Chat Completions не должен закрывать turn
**Решение:**
- Если Chat Completions response содержит и assistant text, и tool calls, synthesized assistant message должен иметь `end_turn: false`.
- `end_turn: true` остаётся только для assistant text без tool calls.
- Regression test должен покрывать и structured `tool_calls`, и legacy `<tool_call><function_call>...` markup.

**Причина:**
- Реальный LLMOps/Gena flow может вернуть prelude text и tool call в одном Chat Completions response.
- `end_turn: true` на prelude создаёт ложную turn boundary перед tool result и может ломать continuation/interruption semantics.

**Подтверждение:**
- Реализовано в рабочем дереве `gena-rs-project`.
- Проверено focused-тестами:
  - `chat_completion_text_before_tool_call_does_not_end_turn`
  - `chat_completion_text_before_legacy_tool_call_does_not_end_turn`
  - `chat_completions_text_before_tool_call_runs_tool_loop_to_completion`

**Альтернативы:**
- Всегда ставить `end_turn: true` для любого assistant text.
- Не синтезировать assistant item для prelude text.
- Решать это только на уровне TUI rendering.

## 2026-05-04 — Release artifacts пересобраны до ручного TUI smoke по явному запросу
**Решение:**
- Пересобрать `gena v0.125.0` release artifacts после focused tests, debug build и non-interactive LLMOps smoke.
- Сохранить в `NOW.md` факт, что manual TUI smoke всё ещё не закрыт.

**Причина:**
- Пользователь явно попросил продолжить и не забыть собрать release `gena`.
- Debug/non-interactive path дал полезную проверку hotfix, но manual TUI gate остаётся отдельным user-visible сценарным gate.

**Подтверждение:**
- Пересобраны:
  - `dist/gena-v0.125.0-macos-arm64.tar.gz`
  - `dist/gena-v0.125.0-macos-arm64.tar.gz.sha256`
  - `dist/gena-v0.125.0-macos-arm64-installer.sh`
- Artifact versions:
  - `gena 0.125.0`
  - `codex-tui 0.125.0`

**Альтернативы:**
- Строго остановиться до ручного TUI smoke.
- Не собирать release до отдельного commit.
- Собирать только debug и оставить release на следующую сессию.
