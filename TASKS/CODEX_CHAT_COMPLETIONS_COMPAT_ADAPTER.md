# ROADMAP: Chat Completions Adapter для Responses-native agent loop в Gena CLI

Источник: `TASKS/CODEX_CHAT_COMPLETIONS_COMPAT_ADAPTER.md`

Цель: реализовать Chat Completions compatibility adapter для Responses-native agent loop в Gena CLI без отдельного agent loop и без поломки `WireApi::Responses`.

## Главный архитектурный выбор

Для Gena основное и единственное решение в рамках этой задачи — **internal adapter внутри core**.

**Shim/proxy не реализовывать.**

```text
Gena Agent Loop
  работает только с ResponseItem / ResponseEvent
        |
        v
ModelClient
  ├── ResponsesAdapter
  │     -> /v1/responses
  │
  └── ChatCompletionsAdapter
        -> /v1/chat/completions
```

Главное правило:

```text
Agent loop не знает, что под капотом был Chat Completions.
Chat Completions Adapter делает вид, что Chat Completions — это Responses API.
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

Почему:

- внешний shim/proxy не видит внутренние `ResponseItem` / `ResponseEvent` так же удобно, как core;
- внешний shim/proxy хуже понимает tool lifecycle;
- усложняется отладка agent loop;
- появляется лишний сетевой слой;
- сложнее тестировать full loop;
- выше риск расхождения с upstream Codex;
- это уводит задачу от нужной архитектуры.

## Почему нужен именно adapter

Adapter находится внутри runtime и видит:

- `ResponseItem`;
- `ResponseEvent`;
- `Prompt`;
- tool definitions;
- tool results;
- turn boundary;
- retries/errors;
- telemetry;
- token usage;
- provider config.

Поэтому adapter может корректно мапить Chat Completions в Responses-style agent loop без отдельного loop и без отдельного tool executor.

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
codex-rs/model-provider-info/src/lib.rs
codex-rs/core/src/client.rs
codex-rs/core/src/chat_completions_adapter.rs
codex-rs/codex-api/src/common.rs
codex-rs/codex-api/src/endpoint/chat_completions.rs
codex-rs/core/tests/suite/client.rs
```

## Цель

Сделать так, чтобы при:

```bash
LLMOPS_WIRE_API=chat-completions
```

agent loop Gena/Codex мог нормально:

1. отправить запрос в Chat Completions API;
2. получить assistant text;
3. получить tool calls;
4. выполнить tools через существующий tool executor;
5. отправить tool results обратно в модель;
6. получить финальный ответ;
7. не ломать Responses-native режим.

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

# Phase 1 — Wire API Routing

1. Добавить явную развилку по `self.state.provider.info().wire_api`.
2. Оставить существующий path для `WireApi::Responses` без изменения поведения.
3. Ограничить WebSocket path только `WireApi::Responses`.
4. Для `WireApi::ChatCompletions` направить выполнение в `ChatCompletionsAdapter` через HTTP unary request.
5. Не добавлять внешний shim/proxy.
6. Добавить regression coverage, подтверждающую, что Responses path не ушёл в Chat Completions branch.

Целевая форма:

```rust
match self.state.provider.info().wire_api {
    WireApi::Responses => {
        // existing Responses path
    }
    WireApi::ChatCompletions => {
        // ChatCompletionsAdapter path
    }
}
```

# Phase 2 — Adapter Skeleton

1. Создать отдельный модуль adapter:

```text
codex-rs/core/src/chat_completions_adapter.rs
```

2. Adapter должен быть внутренним core-компонентом, а не HTTP shim/proxy.
3. Реализовать entrypoint уровня:

```rust
pub(crate) async fn stream_chat_completions_as_responses(
    client_setup: CurrentClientSetup,
    prompt: &Prompt,
    model_info: &ModelInfo,
    telemetry: ...,
) -> Result<ResponseStream>
```

Сигнатуру адаптировать под реальные visibility и текущие структуры.

4. Adapter должен:
   - построить `ChatCompletionsRequest`;
   - вызвать `ChatCompletionsClient::complete`;
   - преобразовать результат в Responses-style events;
   - вернуть `ResponseStream`.
5. Для non-stream Chat Completions эмулировать event stream через `mpsc::channel`.
6. Не делать fake token-by-token streaming; отправлять последовательные semantic events.
7. Не создавать отдельный agent loop внутри adapter.
8. Не создавать отдельный tool executor внутри adapter.

# Phase 3 — Input Mapping

1. Реализовать converter:

```rust
fn responses_input_to_chat_messages(
    instructions: &str,
    input: &[ResponseItem],
) -> Vec<ChatCompletionsRequestMessage>
```

2. Mapping minimum:
   - instructions -> `system`;
   - user input -> `user`;
   - assistant text -> `assistant`;
   - assistant tool call, если нужен для истории -> `assistant` with `tool_calls`;
   - tool output -> `tool` message with `tool_call_id`.
3. При необходимости расширить `ChatCompletionsRequestMessage` полями:
   - `tool_call_id`;
   - `name`;
   - `tool_calls`, если текущий формат требует сохранять assistant tool calls в истории.
4. Сохранить OpenAI-compatible формат tool-result message:

```json
{
  "role": "tool",
  "tool_call_id": "call_xxx",
  "content": "..."
}
```

5. Не терять связку:

```text
assistant tool_call id -> tool result tool_call_id
```

6. Добавить unit tests:
   - instructions + user input -> system + user messages;
   - assistant text ResponseItem -> assistant message;
   - tool output ResponseItem -> role `tool` with `tool_call_id`;
   - tool result сохраняет call id.

# Phase 4 — Tool Schema Mapping

1. Переиспользовать существующую генерацию tool schema, если формат совместим.
2. Если Responses-tools несовместимы с Chat Completions, добавить маленький converter:

```rust
fn responses_tools_to_chat_tools(tools: Vec<Value>) -> Vec<Value>
```

3. Целевой формат tool schema:

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

4. Не дублировать tool definitions и не менять Responses tool schema.
5. Не hardcode-ить tools только под `shell` или только под `llmops`.
6. Добавить unit tests на Chat Completions tool schema format.

# Phase 5 — Output Mapping

1. Реализовать mapping:

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

2. Реализовать mapping:

```text
ChatCompletionsToolCall -> ResponseItem tool/function call
```

3. Сохранять `call_id`, если он есть.
4. Если `id` отсутствует, генерировать стабильный id вида:

```text
chatcmpl-call-{index}
```

5. Передавать `arguments` как JSON string без поломки escaping.
6. Если `arguments` невалидный JSON, не падать в adapter; передать дальше и дать loop/tool executor обработать ошибку.
7. Adapter не должен сам выполнять tool call.
8. Добавить unit tests:
   - Chat tool call -> ResponseItem tool call;
   - multiple tool calls -> multiple ResponseItem tool calls;
   - missing id -> generated stable id;
   - invalid arguments do not crash adapter.

# Phase 6 — Usage And Turn Boundary

1. Реализовать mapping:

```text
ChatCompletionsResponseUsage -> TokenUsage
```

2. Заполнять только доступные поля, остальные оставлять default/None.
3. Передавать usage в `ResponseEvent::Completed`.
4. Правило `end_turn`:
   - no tool calls -> `Some(true)`;
   - has tool calls -> `Some(false)`.
5. Смысл правила: если модель вернула tool calls, turn ещё не финализирован и agent loop должен продолжить через tool executor.
6. Добавить unit tests:
   - usage mapping;
   - no tool calls -> end_turn true;
   - has tool calls -> end_turn false.

# Phase 7 — Reasoning Downgrade

1. Для `WireApi::ChatCompletions` не отправлять Responses-only controls:
   - `reasoning`;
   - `text.verbosity`, если Chat API не поддерживает;
   - `include`;
   - `store`;
   - `parallel_tool_calls`, если provider может не поддерживать;
   - Responses WebSocket metadata.
2. Если в config/model_info задан reasoning effort/summary, adapter не должен падать.
3. Добавить `tracing::debug!` о graceful downgrade без пользовательского шума:

```rust
tracing::debug!("reasoning controls ignored for chat-completions wire api");
```

4. Добавить coverage на сценарий с заданным reasoning config в Chat Completions mode.

# Phase 8 — No Shim/Proxy

1. Не реализовывать внешний shim.
2. Не реализовывать внешний proxy.
3. Не добавлять зависимость runtime от промежуточного HTTP-сервиса.
4. Основной runtime path должен быть только такой:

```text
Gena core -> ChatCompletionsAdapter -> ChatCompletionsClient -> provider /v1/chat/completions
```

# Phase 9 — Mock Integration Tests

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

# Phase 10 — Validation

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
12. Adapter изолирован в отдельном модуле или clearly separated block и не размывает `ModelClient`.

# Constraints

1. Не делать отдельный agent loop для Chat Completions.
2. Не делать отдельный tool executor для Chat Completions.
3. Не hardcode-ить реализацию только под `llmops`.
4. Не ломать Responses API path.
5. Не смешивать provider config и runtime loop logic.
6. Держать diff минимальным относительно upstream Codex.
7. Делать mapping маленькими отдельными функциями с тестами.
8. Не добавлять shim/proxy в рамках этой задачи.

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

Главное правило:

```text
Agent loop не должен знать, что под капотом был Chat Completions.
```
