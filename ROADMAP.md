# gena-rs — ROADMAP

Версия: v1.0
Формат: CLI + Plugin Platform
Статус: основной концептуальный документ проекта

Цель документа — зафиксировать направления развития `gena-rs`
как CLI-агента и plugin platform, построенных поверх upstream `Codex`.

---

## 1. Upstream sync strategy

**Цель:** поддерживать upstream `Codex` актуальным при минимальном merge pressure.

### Подход
- Upstream-owned zone максимально близка к оригиналу
- Gena-specific логика не размазывается по upstream crates
- После каждого merge: локализовать compile/runtime breakage в `gena-upstream-adapter`

### Операционное правило (OAuth-safe push)
- При push через OAuth remote без `workflow` scope: дропать `.github/workflows/*` follow-up commit'ом перед push
- Commit message: `sync(upstream): drop workflow changes for oauth push`

Результат: стабильный sync-процесс с минимальной ручной правкой.

---

## 2. Branding layer

**Цель:** централизовать все product-specific user-visible строки в `gena-branding`.

### Что относится
- product name, binary names
- CLI help copy, banner/splash
- config folder names, session folder names
- env prefixes, docs references

### Принцип
Бренд должен инжектироваться, а не быть рассыпанным точечными `if gena`.

Результат: `gena-branding` crate как единственный источник истины по бренду.

---

## 3. Runtime boundaries

**Цель:** выделить `gena-runtime` как отдельный crate с чёткими ответственностями.

### Ответственности runtime
- session lifecycle
- orchestration
- tool execution pipeline
- workflow coordination
- plugin loader integration
- permissions / policies
- runtime state
- tracing / observability

### Текущий checkpoint (architecturally complete)
- Gena-owned runtime/platform layers собраны
- Plugin platform имеет: capability axes, execution stack, lifecycle layer, dynamic loader, registry/distribution surface
- `gena-runtime` нормализован в отдельные boundary-модули

Результат: `gena-runtime` с минимальным structural pressure и явными crate-границами.

---

## 4. Plugin platform

**Цель:** стабильный plugin ecosystem с полным lifecycle management.

### Текущие возможности
- Static discovery + dynamic loader
- Compatibility manifest validation
- Bundle convention (`gena-plugin.toml`)
- Archive package convention (`gena-package.toml`)
- Local install / replace / uninstall loop
- Remote archive URL fetch/install
- Registry/index inspect, search (fuzzy + ranked), install
- Service-backed registry (cursor pagination, auth-aware)

### Следующие фронты
- `gena-plugin-api` стабилизация
- `gena-plugins-core` перенос built-in features
- Внешние plugins в `plugins/` директории

Результат: production-ready plugin platform с registry-backed distribution.

---

## 5. Config layer

**Цель:** отделить загрузку/нормализацию конфигурации от runtime-логики.

### Что относится
- config schema, defaults, env overrides, CLI overrides
- file loading, profile handling
- compatibility migrations

Результат: `gena-config` crate без размазанной по runtime config-логики.

---

## 6. Adapter layer

**Цель:** изолировать `gena-runtime` от деталей upstream `Codex`.

### Ответственности
- backend adapters, conversion layer
- compatibility helpers, feature probing
- bridge к upstream behavior
- первая точка ремонта после upstream API drift

Результат: `gena-upstream-adapter` как единственная зона починки после upstream update.

---

## 7. Правила для агента (в контексте gena-rs)

**Execution rules**
- после изменений Rust-кода всегда запускать `just fmt` без запроса
- перед финализацией крупных изменений запускать `just fix -p <crate>`
- полный `cargo test` только с явного разрешения пользователя
- никогда не трогать `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` и `CODEX_SANDBOX_ENV_VAR`

**Reasoning rules**
- не принимать архитектурных решений без фиксации в DL.md
- не делать тотальный rewrite repo раньше, чем выделены реальные architectural seams

**UX rules**
- snapshot-тесты (insta) обязательны для любых UI-изменений в TUI
- docs в `docs/` обновлять при изменении API

---

## Итог

gena-rs — это Codex-based CLI-платформа с собственным:

- plugin ecosystem
- branding layer
- config layer
- upstream adapter

Стратегия: additive migration поверх upstream layout, без тотального rewrite.

Этот roadmap является опорным документом для дальнейшей разработки.
