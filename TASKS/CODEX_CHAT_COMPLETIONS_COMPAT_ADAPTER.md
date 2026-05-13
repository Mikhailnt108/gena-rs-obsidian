# ROADMAP: Chat Completions Adapter для Responses-native agent loop в Gena CLI

Источник: обсуждение архитектуры Gena CLI / Codex upstream sync.

Цель: реализовать Chat Completions compatibility adapter для Responses-native agent loop в Gena CLI без отдельного agent loop, без shim/proxy, без поломки `WireApi::Responses` и с минимальным diff относительно upstream Codex.

## Главный архитектурный выбор

Для Gena решение — **internal Rust adapter**, а не внешний HTTP shim/proxy.

Предпочтительная целевая форма — отдельный workspace crate:

```text
codex-rs/
  gena-chat-completions-adapter/
    Cargo.toml
    src/
      lib.rs
      input_mapping.rs
      output_mapping.rs
      tool_mapping.rs
      usage_mapping.rs
      stream_emulation.rs
      tests.rs
```

Runtime path:

```text
Gena Agent Loop
  работает только с ResponseItem / ResponseEvent
        |
        v
codex-core / ModelClient
  ├── Responses path
  │     -> /v1/responses
  │
  └── thin WireApi routing
        -> gena-chat-completions-adapter
             -> ChatCompletionsClient
                  -> provider /v1/chat/completions
```

Главное правило:

```text
Agent loop не знает, что под капотом был Chat Completions.
Chat Completions Adapter делает вид, что Chat Completions — это Responses API.
```

## Crate-first, module-fallback стратегия

Нужно стремиться к отдельному crate:

```text
gena-chat-completions-adapter
```

Почему crate лучше:

- лучше изоляция Gena-specific logic;
- меньше конфликтов при upstream update;
- проще contract tests;
- проще увидеть границу adapter-а;
- меньше риск размазать mapping по `codex-core`;
- соответствует текущему направлению workspace, где уже есть `gena-*` crates.

Но если отдельный crate требует большого раскрытия приватных upstream API (`Prompt`, `CurrentClientSetup`, `ResponseStream` и т.д.), стартовать можно как crate-ready module внутри `codex-core`:

```text
codex-rs/core/src/chat_completions_adapter/
  mod.rs
  input_mapping.rs
  output_mapping.rs
  tool_mapping.rs
  usage_mapping.rs
  stream_emulation.rs
  tests.rs
```

Требование к module fallback:

```text
Писать его так, чтобы позже вынести в отдельный crate без переписывания архитектуры.
```

Итоговая стратегия:

```text
1. Попробовать отдельный crate `gena-chat-completions-adapter`.
2. Если это резко увеличивает public API/diff с upstream, стартовать с `codex-core/src/chat_completions_adapter/`.
3. Сохранить crate-ready структуру и explicit TODO/roadmap на вынос в crate.
```

## Что НЕ делать

Не делать runtime-схему:

```text
Gena -> fake /v1/responses shim -> provider /v1/chat/completions
```

Не делать runtime-схему:

```text
Gena -> external proxy -> provider /v1/chat/completions
```

Не делать:

```text
отдельный Chat Completions agent loop
отдельный Chat Completions tool executor
hardcode только под llmops
размазывание mapping по client.rs / loop / tool executor
```

## Upstream-safe design

Так как Gena регулярно обновляет upstream Codex, adapter должен быть реализован так, чтобы после апдейтов upstream не приходилось бесконечно чинить mapping и конфликты.

Главный принцип:

```text
Минимальная точка врезки в upstream-код + изолированный crate/module + contract tests.
```

Правильная форма:

```text
upstream-like ModelClient
  └── маленькая развилка по WireApi
        ├── existing Responses path untouched
        └── call into Gena-owned adapter boundary
```

Неправильная форма:

```text
размазать Chat Completions mapping по client.rs / loop / tool executor / protocol handling
```

### Правила upstream-safe реализации

1. `WireApi::Responses` path не менять, кроме минимальной развилки, если это технически необходимо.
2. В `codex-rs/core/src/client.rs` держать только thin routing layer.
3. Всю Chat Completions compatibility logic держать в `gena-chat-completions-adapter` crate или в crate-ready module fallback.
4. Не менять существующий tool executor.
5. Не менять существующий Responses event loop.
6. Не менять существующий Responses WebSocket path.
7. Не добавлять Gena-specific логику в upstream-like участки без необходимости.
8. Если требуется изменить общий тип (`ChatCompletionsRequestMessage`, `ChatCompletionsOutput`, `ResponseEvent`), изменение должно быть минимальным, generic и покрыто тестами.
9. Adapter должен зависеть от stable boundary types:
   - `Prompt`;
   - `ResponseItem`;
   - `ResponseEvent`;
   - `ResponseStream`;
   - `ToolSpec` / tool schema;
   - `TokenUsage`.
10. Не завязывать adapter на внутренние детали конкретного provider, например только `llmops`.
11. Все mapping-функции должны быть маленькими и покрыты unit tests.

### Целевой diff после upstream update

После обновления upstream Codex ожидаемый repair surface должен быть таким:

```text
1. Проверить, что thin WireApi routing в ModelClient ещё компилируется.
2. Проверить, что adapter entrypoint ещё получает нужные stable boundary types.
3. Запустить contract tests mapping-функций.
4. Если upstream поменял ResponseItem/ResponseEvent, чинить только adapter mapping, а не весь loop.
```

Цель — чтобы upstream update обычно требовал правки только в:

```text
gena-chat-completions-adapter/*
```

или во временном fallback:

```text
codex-rs/core/src/chat_completions_adapter/*
```

а не в:

```text
agent loop
tool executor
Responses stream handling
WebSocket handling
multiple unrelated upstream files
```

## Что уже есть

В проекте уже есть:

- `WireApi::Responses`;
- `WireApi::ChatCompletions`;
- built-in provider `llmops`;
- env-переключатель `LLMOPS_WIRE_API`;
- `ChatCompletionsClient`;
- базовый парсинг:
  - `content`;
  - `tool_calls`;
  - legacy `function_call`;
  - `usage`.

Ключевые файлы-кандидаты:

```text
codex-rs/Cargo.toml
codex-rs/model-provider-info/src/lib.rs
codex-rs/core/src/client.rs
codex-rs/gena-chat-completions-adapter/*
codex-rs/core/src/chat_completions_adapter/*  # fallback only
codex-rs/codex-api/src/common.rs
codex-rs/codex-api/src/endpoint/chat_completions.rs
codex-rs/core/tests/suite/client.rs
```

---

# Phase 0 — Baseline Discovery

1. Найти текущую точку выбора `wire_api` в `codex-rs/core/src/client.rs`.
2. Зафиксировать текущий Responses request/stream path, включая HTTP Responses и WebSocket.
3. Найти существующие типы и variants для:
   - `ResponseItem` function/tool call;
   - `ResponseItem` tool result / function output;
   - `ResponseEvent::OutputItemDone`;
   - `TokenUsage`;
   - `ResponseStream`.
4. Проверить текущие структуры Chat Completions в:
   - `codex-rs/codex-api/src/common.rs`;
   - `codex-rs/codex-api/src/endpoint/chat_completions.rs`.
5. Найти существующую генерацию tool schema для Responses API и оценить совместимость с Chat Completions.
6. Проверить, как current loop добавляет tool result обратно в следующий model request.
7. Проверить, какие boundary types можно использовать из отдельного crate без большого раскрытия private API.
8. Принять решение:
   - `gena-chat-completions-adapter` crate сразу;
   - или `codex-core/src/chat_completions_adapter/` как временный crate-ready fallback.

# Phase 1 — Crate / Module Boundary

1. Предпочтительно создать workspace crate:

```text
codex-rs/gena-chat-completions-adapter
```

2. Добавить его в `codex-rs/Cargo.toml` workspace members и workspace dependencies.
3. Подключить crate в `codex-core` как dependency.
4. Если отдельный crate требует большого раскрытия приватных API, создать временный module fallback:

```text
codex-rs/core/src/chat_completions_adapter/
```

5. Module fallback должен повторять будущую crate-структуру:

```text
mod.rs
input_mapping.rs
output_mapping.rs
tool_mapping.rs
usage_mapping.rs
stream_emulation.rs
tests.rs
```

6. Не смешивать adapter mapping с `client.rs`.

# Phase 2 — Wire API Routing

1. Добавить явную развилку по `self.state.provider.info().wire_api`.
2. Оставить существующий path для `WireApi::Responses` без изменения поведения.
3. Ограничить WebSocket path только `WireApi::Responses`.
4. Для `WireApi::ChatCompletions` направить выполнение в adapter через HTTP unary request.
5. Не добавлять внешний shim/proxy.
6. Добавить regression coverage, подтверждающую, что Responses path не ушёл в Chat Completions branch.
7. Держать routing-код максимально тонким: route decision + вызов adapter entrypoint.

Целевая форма:

```rust
match self.state.provider.info().wire_api {
    WireApi::Responses => {
        // existing Responses path
    }
    WireApi::ChatCompletions => {
        // Gena adapter path
    }
}
```

# Phase 3 — Adapter Skeleton

Adapter entrypoint должен:

1. построить `ChatCompletionsRequest`;
2. вызвать `ChatCompletionsClient::complete`;
3. преобразовать результат в Responses-style events;
4. вернуть `ResponseStream`.

Примерная форма:

```rust
pub async fn stream_chat_completions_as_responses(
    input: AdapterInput,
) -> Result<ResponseStream>
```

Где `AdapterInput` — explicit boundary struct, чтобы не протаскивать через crate много приватных типов.

Требования:

- для non-stream Chat Completions эмулировать event stream через `mpsc::channel`;
- не делать fake token-by-token streaming;
- не создавать отдельный agent loop;
- не создавать отдельный tool executor;
- не подтягивать зависимости от TUI/UI/CLI;
- mapping-функции делать pure, где возможно.

# Phase 4 — Input Mapping

Реализовать converter:

```rust
fn responses_input_to_chat_messages(
    instructions: &str,
    input: &[ResponseItem],
) -> Vec<ChatCompletionsRequestMessage>
```

Mapping minimum:

- instructions -> `system`;
- user input -> `user`;
- assistant text -> `assistant`;
- assistant tool call, если нужен для истории -> `assistant` with `tool_calls`;
- tool output -> `tool` message with `tool_call_id`.

OpenAI-compatible tool-result message:

```json
{
  "role": "tool",
  "tool_call_id": "call_xxx",
  "content": "..."
}
```

Unit tests:

- instructions + user input -> system + user messages;
- assistant text ResponseItem -> assistant message;
- tool output ResponseItem -> role `tool` with `tool_call_id`;
- tool result сохраняет call id.

# Phase 5 — Tool Schema Mapping

1. Переиспользовать существующую генерацию tool schema, если формат совместим.
2. Если Responses-tools несовместимы с Chat Completions, добавить маленький converter:

```rust
fn responses_tools_to_chat_tools(tools: Vec<Value>) -> Vec<Value>
```

Целевой формат:

```json
{
  "type": "function",
  "function": {
    "name": "...",
    "description": "...",
    "parameters": { }
  }
}
```

Требования:

- не дублировать tool definitions;
- не менять Responses tool schema;
- не hardcode-ить tools только под `shell` или только под `llmops`;
- добавить unit tests на Chat Completions tool schema format.

# Phase 6 — Output Mapping

Реализовать mapping:

```text
ChatCompletionsOutput -> ResponseEvent stream
```

Минимальная последовательность:

```text
ResponseEvent::Created
ResponseEvent::OutputTextDelta(...), если есть content
ResponseEvent::OutputItemDone(...), если есть assistant text item
ResponseEvent::OutputItemDone(...), для каждого tool call item
ResponseEvent::Completed { ... }
```

Реализовать mapping:

```text
ChatCompletionsToolCall -> ResponseItem tool/function call
```

Требования:

- сохранять `call_id`, если он есть;
- если `id` отсутствует, генерировать стабильный id `chatcmpl-call-{index}`;
- передавать `arguments` как JSON string без поломки escaping;
- если `arguments` невалидный JSON, не падать в adapter;
- adapter не должен сам выполнять tool call.

Unit tests:

- Chat tool call -> ResponseItem tool call;
- multiple tool calls -> multiple ResponseItem tool calls;
- missing id -> generated stable id;
- invalid arguments do not crash adapter.

# Phase 7 — Usage And Turn Boundary

Реализовать mapping:

```text
ChatCompletionsResponseUsage -> TokenUsage
```

Правило `end_turn`:

- no tool calls -> `Some(true)`;
- has tool calls -> `Some(false)`.

Unit tests:

- usage mapping;
- no tool calls -> end_turn true;
- has tool calls -> end_turn false.

# Phase 8 — Reasoning Downgrade

Для `WireApi::ChatCompletions` не отправлять Responses-only controls:

- `reasoning`;
- `text.verbosity`, если Chat API не поддерживает;
- `include`;
- `store`;
- `parallel_tool_calls`, если provider может не поддерживать;
- Responses WebSocket metadata.

Если в config/model_info задан reasoning effort/summary, adapter не должен падать.

Добавить debug trace:

```rust
tracing::debug!("reasoning controls ignored for chat-completions wire api");
```

# Phase 9 — No Shim/Proxy

1. Не реализовывать внешний shim.
2. Не реализовывать внешний proxy.
3. Не добавлять зависимость runtime от промежуточного HTTP-сервиса.
4. Основной runtime path должен быть только такой:

```text
Gena core -> adapter crate/module -> ChatCompletionsClient -> provider /v1/chat/completions
```

# Phase 10 — Upstream-safe Boundary Hardening

1. Проверить, что изменения в upstream-like `client.rs` сведены к минимуму.
2. Если возможно, оставить в `client.rs` только:
   - import adapter crate/module;
   - match по `WireApi`;
   - вызов adapter entrypoint.
3. Вынести все mapping-функции из `client.rs`.
4. Добавить комментарий рядом с routing-точкой:

```rust
// Gena: keep Chat Completions compatibility isolated in adapter crate/module
// to reduce upstream merge conflicts. Do not add mapping logic here.
```

5. Добавить contract tests на adapter boundary:
   - adapter принимает canonical input;
   - adapter возвращает canonical `ResponseStream` events;
   - Responses path не зависит от adapter.
6. Если upstream меняет `ResponseItem` / `ResponseEvent`, expected repair должен быть только в adapter mapping tests.

# Phase 11 — Mock Integration Tests

1. Добавить mock e2e test: simple final answer.
2. Добавить mock e2e test: assistant returns tool call.
3. Добавить mock e2e test full loop:
   - model returns tool call;
   - existing tool executor runs tool;
   - tool result goes into next model request;
   - model returns final answer.
4. Проверить, что Chat Completions branch не создает отдельный tool executor.
5. Проверить, что agent loop не знает о Chat Completions internals.
6. Проверить, что второй request после tool execution содержит корректный `role=tool` / `tool_call_id`.
7. Добавить regression test, который падает, если Chat Completions path начинает использовать Responses WebSocket.

# Phase 12 — Validation

1. Запустить после Rust-изменений:
   - `just fmt`;
   - targeted unit/integration tests for touched crates.
2. Перед финализацией крупного изменения запустить scoped:
   - `just fix -p <crate>`.
3. Полный `cargo test` запускать только с явного разрешения пользователя.
4. Manual LLMOps validation:
   - simple prompt without tools;
   - read file task;
   - create file task;
   - edit file task;
   - shell command task.
5. После следующего upstream update отдельно проверить:
   - компиляцию routing-точки;
   - adapter mapping tests;
   - full-loop mock test.

# Acceptance Criteria

1. `WireApi::Responses` работает как раньше.
2. `WireApi::ChatCompletions` не падает на обычном запросе.
3. Chat Completions может вернуть финальный assistant answer.
4. Chat Completions может вернуть tool call.
5. Существующий agent loop выполняет tool call.
6. Tool result уходит обратно в модель.
7. После tool result модель может вернуть final answer.
8. Responses-specific reasoning/WebSocket logic не вызывается для Chat Completions.
9. Есть unit tests на mapping.
10. Есть хотя бы один mock integration test на full loop: `assistant tool_call -> tool result -> assistant final`.
11. В runtime нет внешнего shim/proxy.
12. Adapter изолирован в отдельном crate или crate-ready module fallback.
13. Изменения в upstream-like files минимальны и локализованы.
14. После upstream update ожидаемая зона ремонта — adapter mapping, а не agent loop/tool executor.
15. Есть tests, которые защищают boundary adapter-а.
16. Если adapter стартует как module fallback, есть явный путь миграции в `gena-chat-completions-adapter` crate.

# Constraints

1. Не делать отдельный agent loop для Chat Completions.
2. Не делать отдельный tool executor для Chat Completions.
3. Не hardcode-ить реализацию только под `llmops`.
4. Не ломать Responses API path.
5. Не смешивать provider config и runtime loop logic.
6. Держать diff минимальным относительно upstream Codex.
7. Делать mapping маленькими отдельными функциями с тестами.
8. Не добавлять shim/proxy в рамках этой задачи.
9. Не размазывать Chat Completions logic по upstream-like файлам.
10. Не менять upstream-like код там, где достаточно adapter boundary.
11. Не раскрывать массово private upstream API только ради crate; если требуется слишком большой public API diff, использовать crate-ready module fallback.

# Desired final architecture

```text
ModelClient
  ├── Responses path
  │     ├── HTTP Responses
  │     └── Responses WebSocket
  │
  └── ChatCompletionsAdapter path
        ├── ResponseItem[] -> Chat messages[]
        ├── ToolSpec[] -> chat tools[]
        ├── Chat output -> ResponseEvent
        └── Chat tool_calls -> existing tool executor
```

Preferred code boundary:

```text
gena-chat-completions-adapter crate
```

Allowed temporary fallback:

```text
codex-core/src/chat_completions_adapter/ with crate-ready structure
```

Главное правило:

```text
Agent loop не должен знать, что под капотом был Chat Completions.
```

Upstream-safe правило:

```text
После обновления upstream Codex чинить нужно adapter boundary, а не весь agent loop.
```
