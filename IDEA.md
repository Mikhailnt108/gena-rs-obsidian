# IDEA

## Кратко
`gena-rs` — production CLI-агент и plugin platform, построенные поверх upstream `Codex (codex-rs)`.
Главная задача: дать Gena собственный runtime, plugin ecosystem и branding при минимальном drift от upstream.

## Архитектурная схема
- `upstream-owned zone` — upstream `Codex` crates, максимально близко к оригиналу.
- `gena-upstream-adapter` — изоляция от деталей Codex, локализация API drift.
- `gena-runtime` — ядро поведения Gena: session lifecycle, orchestration, plugin loader, tool execution.
- `gena-plugin-api` — стабильный контракт для расширений (traits, metadata, registration).
- `gena-plugins-core` — встроенные first-party plugins.
- `gena-branding` — все product-specific user-visible identity values.
- `gena-config` — загрузка и нормализация конфигурации.
- `gena-types` — общие типы между Gena crates.
- `gena-cli` — верхний wiring/bootstrap слой.

### Поток зависимостей
```
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

## Принципы
- upstream максимально чистый, Gena-логика не размазывается по upstream-зонам
- branding централизован в одном crate, не точечные `if gena`
- features растут через стабильные plugin contracts
- cheap upstream sync важнее идеальной архитектурной красоты
- additive migration: сначала добавляем Gena-owned boundaries рядом с upstream layout, потом решаем про физический перенос

## Роль агента
Агент исполняет изменения, опираясь на память Obsidian и правила процесса. Код — результат, Obsidian — процесс.
