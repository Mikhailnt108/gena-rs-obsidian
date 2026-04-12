# GENA_REPO_STRUCTURE

## Назначение

Этот документ фиксирует целевую архитектуру и структуру workspace для `Gena CLI`, построенного поверх upstream `Codex`.

Главная цель не в том, чтобы сделать "красивый новый layout", а в том, чтобы:

- удешевить sync с upstream `Codex`
- минимизировать брендовые правки `Gena`
- минимизировать Gena-specific feature diff внутри upstream-кода
- локализовать ломкость после апдейта upstream
- дать понятные crate-границы для долгой поддержки

Ключевое правило:

> `Gena` не должен жить как грязный fork `Codex`.  
> Upstream-код должен оставаться максимально близким к оригиналу.

## Главный архитектурный вывод

Предлагаемая структура в целом удачная как **target architecture**, но ее нельзя воспринимать как команду на немедленный тотальный filesystem rewrite текущего `codex-rs`.

Правильный подход:

- принять эту структуру как целевую модель
- внедрять ее поэтапно
- не устраивать массовый перенос upstream-crates только ради нового дерева каталогов

Иначе стоимость migration может стать выше, чем выигрыш от самой архитектуры.

## Что в предложенной структуре удачно

Ниже зафиксированы сильные стороны подхода.

### 1. Правильно выделены основные слои

Хорошо, что отдельно выделены:

- `gena-upstream-adapter`
- `gena-runtime`
- `gena-plugin-api`
- `gena-plugins-core`
- `gena-branding`
- `gena-config`
- `gena-types`
- `gena-cli`

Это правильное направление, потому что:

- upstream drift должен оседать в adapter layer
- брендинг не должен течь в upstream-код
- runtime не должен знать детали private internals `Codex`
- расширения должны расти через plugin contracts

### 2. Правильно поставлена цель: cheap upstream sync

Для `Gena` это важнее любой "идеальной" архитектурной красоты.

Если после каждого upstream merge приходится:

- править `core/`
- править `tui/`
- править `cli/`
- заново раскладывать брендовые строки

значит архитектура не выполняет свою задачу.

### 3. Правильно выделен branding как отдельный слой

Это особенно важно для `gena`, потому что сейчас брендовые различия естественно стремятся расползтись по:

- CLI help
- startup UX
- конфиг-путям
- директориям состояния
- alias-логике бинарей

Если это не централизовать, каждый upstream sync будет конфликтнее, чем должен быть.

### 4. Правильная ставка на plugin API

Путь через плагины хорош для:

- интеграций
- domain-specific automation
- memory backends
- provider extensions
- workflow extensions

Это помогает не раздувать runtime и не тащить feature-specific код в upstream-shaped зону.

## Что в предложенной структуре нужно скорректировать

Вот ключевые уточнения, без которых структура может оказаться слишком идеалистичной и дорогой в реальном форке.

### 1. `codex-upstream` не стоит понимать буквально как один новый crate

Если текущий upstream уже живет как workspace из множества crates, то попытка собрать его в один физический `crates/codex-upstream/` даст лишний шум:

- большой rename / move diff
- тяжелые merge-конфликты
- дорогой rebase/sync
- ухудшение читаемости истории

Поэтому правильнее мыслить так:

- `codex-upstream` — это **upstream-owned zone**
- она может состоять из набора crates в их близком к upstream layout
- физический перенос допустим только там, где это реально уменьшает стоимость поддержки

Иными словами:

> `codex-upstream` — это логическая зона владения, а не обязательно один crate или один каталог.

### 2. Нельзя делать полный rewrite дерева сразу

Если резко перевести весь репозиторий в новый layout:

- мы сами создадим огромный diff
- ухудшим sync с upstream ровно в тот момент, когда хотим его упростить

Поэтому нужно разделять:

- **target structure**
- **migration strategy**

Target structure может быть амбициозной.  
Migration strategy должна быть консервативной.

### 3. `gena-cli` не должен схлопывать существующие entrypoints слишком рано

Если в текущем состоянии уже есть отдельные зоны вроде:

- `cli`
- `tui`
- `exec`

то не надо насильно сводить все в один новый crate на первом этапе.

Лучше думать так:

- `gena-cli` — это будущий композиционный entrypoint слой
- но миграция туда должна быть постепенной
- сначала нужно стабилизировать branding/config/runtime boundaries

### 4. Плагинную архитектуру нельзя вводить "слишком рано"

Есть риск сделать красивый `gena-plugin-api`, не имея еще устойчивых runtime boundaries.

Тогда получится:

- много абстракций
- мало реальной изоляции
- скрытая связность все равно останется в runtime/upstream code

Поэтому plugin API нужно строить после того, как:

- branding централизован
- config отделен
- adapter layer появился
- runtime-грань стала более явной

## Рекомендуемая целевая структура

Как **target architecture** структура ниже хороша и может быть принята:

```text
repo/
├─ Cargo.toml
├─ Cargo.lock
├─ rust-toolchain.toml
├─ README.md
├─ AGENTS.md
├─ docs/
│  ├─ ARCHITECTURE.md
│  ├─ REPO_STRUCTURE.md
│  ├─ UPSTREAM_SYNC.md
│  ├─ PLUGIN_API.md
│  └─ BRANDING.md
├─ crates/
│  ├─ gena-upstream-adapter/
│  ├─ gena-runtime/
│  ├─ gena-plugin-api/
│  ├─ gena-plugins-core/
│  ├─ gena-branding/
│  ├─ gena-config/
│  ├─ gena-cli/
│  └─ gena-types/
├─ plugins/
│  ├─ gena-plugin-mcp/
│  ├─ gena-plugin-android/
│  ├─ gena-plugin-playwright/
│  └─ gena-plugin-memory-fs/
├─ tests/
│  ├─ integration/
│  ├─ fixtures/
│  └─ snapshots/
├─ scripts/
│  ├─ sync_upstream.sh
│  ├─ check_workspace.sh
│  └─ release_local.sh
└─ upstream-owned crates / zone
```

Важно:

- `upstream-owned crates / zone` здесь указана намеренно в общем виде
- не зашиваемся в требование физически пересобрать весь upstream в один каталог немедленно

## Целевые ответственности по crate-границам

### Upstream-owned zone

Назначение:

- хранить upstream-код максимально близко к `Codex`
- быть основной зоной sync с upstream
- не впитывать в себя Gena-specific runtime / branding / plugin logic

Правила:

- не класть сюда branding
- не класть сюда Gena runtime logic
- не класть сюда plugin system
- патчить только по необходимости
- каждый локальный patch boundary документировать отдельно

### `gena-upstream-adapter`

Назначение:

- изолировать `Gena` от деталей `Codex`
- локализовать API drift после upstream update
- держать mapping между upstream types и Gena-owned abstractions

Ответственность:

- backend adapters
- conversion layer
- compatibility helpers
- feature probing
- bridge к upstream behavior

Главное правило:

> Если upstream что-то сломал, первая точка ремонта должна быть здесь.

### `gena-runtime`

Назначение:

- быть основным ядром поведения `Gena`

Ответственность:

- session lifecycle
- orchestration
- tool execution pipeline
- workflow coordination
- agent/team coordination
- event bus
- memory lifecycle hooks
- permissions / policies
- runtime state
- tracing / observability integration
- plugin loader integration

Главное правило:

- runtime не должен знать private internals `Codex`
- runtime должен зависеть от адаптеров и Gena-owned abstractions

### `gena-plugin-api`

Назначение:

- определить маленький и стабильный контракт для расширений

Содержит:

- core traits
- plugin metadata types
- registration contracts
- extension context types
- легкие shared result/error abstractions

Примеры категорий:

- tools
- providers
- memory
- workflows
- agents
- commands

Правило:

> Этот crate должен быть маленьким, стабильным и не тянуть implementation-heavy зависимости.

### `gena-plugins-core`

Назначение:

- хранить встроенные first-party plugins, поставляемые вместе с `Gena`

Примеры:

- built-in tools
- built-in workflows
- default memory backend
- built-in provider integrations

Правило:

- зависит только от `gena-plugin-api` и публичных extension points runtime
- не лезет в upstream internals напрямую

### `gena-branding`

Назначение:

- держать все product-specific user-visible identity values в одном месте

Сюда относятся:

- product name
- binary names
- CLI help copy
- banner / splash
- config folder names
- session folder names
- env prefixes
- docs references
- branding assets as constants / mappings

Правило:

> Если строка user-visible и product-specific, она должна жить здесь.

### `gena-config`

Назначение:

- отдельно от runtime управлять загрузкой и нормализацией конфигурации

Сюда относятся:

- config schema
- defaults
- env overrides
- CLI overrides
- file loading
- profile handling
- compatibility migrations

Правило:

- config не должен быть размазан по runtime-логике

### `gena-types`

Назначение:

- содержать общие типы, используемые между Gena crates

Примеры:

- session IDs
- request / response DTOs
- runtime enums
- event payloads
- plugin IDs
- provider IDs

Правило:

- crate должен оставаться легким

### `gena-cli`

Назначение:

- быть верхним wiring/bootstrap слоем

Ответственность:

- CLI parsing
- command dispatch
- runtime bootstrap
- branding injection
- configuration bootstrap
- startup wiring

Правило:

- здесь не должно жить глубокое business logic ядра

## Внешняя plugin workspace

Каталог `plugins/` имеет смысл как отдельная зона для:

- optional plugins
- domain-specific integrations
- feature packs, которые не обязаны быть частью core platform

Примеры:

- `gena-plugin-mcp`
- `gena-plugin-android`
- `gena-plugin-playwright`
- `gena-plugin-memory-fs`

Плюсы отдельного каталога:

- чище граница между platform core и integrations
- проще стратегия enable/disable
- прозрачнее ownership boundaries
- проще дальнейшая публикация / versioning

## Правильное направление зависимостей

Правильная зависимость должна выглядеть так:

```text
upstream-owned zone
      ↓
gena-upstream-adapter
      ↓
gena-runtime
      ↓
gena-cli

gena-runtime → gena-plugin-api
gena-plugins-core → gena-plugin-api
external plugins → gena-plugin-api

gena-branding → gena-cli / gena-runtime
gena-config → gena-cli / gena-runtime
gena-types → shared by Gena crates
```

Ключевое правило:

> Зависимости должны смотреть в сторону стабильных абстракций, а не в сторону implementation details.

## Что разрешено и что запрещено

### Разрешено

- `gena-runtime` зависит от `gena-plugin-api`, `gena-types`, `gena-config`
- `gena-cli` зависит от `gena-runtime`, `gena-branding`, `gena-config`
- plugins зависят от `gena-plugin-api` и публичных runtime/plugin context types
- `gena-upstream-adapter` зависит от upstream-owned zone

### Запрещено

- plugins напрямую зависят от upstream-owned crates
- branding код попадает в upstream-owned zone
- runtime глубоко зависит от private structures upstream
- CLI начинает содержать runtime business logic
- `gena-plugin-api` тянет тяжелые implementation crates

## Политика upstream sync

Каждый sync с `Codex` должен следовать такому flow:

1. Подтянуть upstream changes в upstream-owned zone.
2. Посмотреть, какие integration points реально изменились.
3. По возможности чинить compile/runtime breakage в `gena-upstream-adapter`.
4. Прогнать workspace tests.
5. Прогнать plugin compatibility tests.
6. Обновить sync-документацию, если изменились patch boundaries.

Критерий успеха:

- большинство апдейтов не требуют redesign runtime
- большинство фиксов оседают в adapter layer
- branding и Gena-specific features не размазываются обратно в upstream-код

## Политика брендинга

Брендинг должен **инжектиться**, а не быть рассыпанным по коду.

Это касается:

- displayed application name
- help messages
- CLI banner
- binary naming
- config folder naming
- default session folder
- docs references

Правило:

> Бренд должен быть централизованным слоем данных и правил, а не набором точечных условных `if gena`.

## Стратегия тестирования

### Unit tests

Для каждого crate отдельно, с фокусом на:

- adapters
- config
- plugin registration
- runtime orchestration

### Integration tests

Для:

- startup wiring
- plugin loading
- adapter compatibility
- config loading paths

### Snapshot tests

Для:

- CLI output
- branding output
- help output

### Sync tests

Для:

- проверки adapter compatibility после upstream update

## Реалистичный migration plan для текущего `codex-rs`

Ниже самое важное уточнение: для текущего форка нужен не "big-bang rewrite", а **additive migration**.

### Phase 1

- изолировать branding
- изолировать config
- создать adapter crate
- прекратить рост новых Gena-specific features внутри upstream-shaped crates

### Phase 2

- выделить runtime core
- определить plugin API
- перенести built-in Gena features в `gena-plugins-core`

### Phase 3

- вынести domain integrations в `plugins/`
- формализовать upstream sync workflow
- добавить compatibility tests

## Практическое правило миграции

Для текущего `codex-rs` нужно держаться следующего принципа:

> Сначала добавляем новые Gena-owned boundaries рядом с существующим upstream layout.  
> Только потом решаем, нужен ли физический перенос каталогов.

То есть:

- не переносить массово `core/`, `cli/`, `tui/`, `exec/` только ради эстетики
- сначала остановить размазывание Gena-логики по upstream-зонам
- затем постепенно вытягивать branding/config/adapter/runtime в свои crates

Именно такой путь реально уменьшает стоимость upstream sync.

## Итоговый набор правил

1. Держать upstream максимально чистым.
2. Локализовать upstream breakage в adapter layer.
3. Хранить Gena logic в dedicated crates.
4. Держать branding в одном месте.
5. Растить features через стабильные plugin contracts.
6. Следить за чистым направлением зависимостей.
7. Предпочитать явные crate boundaries скрытой связности.
8. Не делать тотальный rewrite repo раньше, чем выделены реальные architectural seams.

## Итог

Целевая формула остается такой:

```text
Codex upstream / upstream-owned zone
        ↓
Gena upstream adapter
        ↓
Gena runtime
        ↓
Gena plugin API
        ↓
Gena plugins / features
```

Итоговая структура полезна как **target architecture**.

Но для текущего `gena-rs` правильная стратегия не в полном мгновенном переписывании дерева, а в постепенном выносе:

- branding
- config
- adapter boundary
- runtime boundary
- plugin system

Только такой путь одновременно:

- упрощает upstream updates
- минимизирует брендовые правки
- минимизирует Gena feature diff
- сохраняет репозиторий сопровождаемым в долгую
