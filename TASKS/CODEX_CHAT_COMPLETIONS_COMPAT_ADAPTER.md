# TASK: Реализовать Chat Completions compatibility adapter для Responses-native agent loop в Gena CLI

## Контекст

Проект: `gena-rs-project`

Gena CLI построен на базе Codex CLI и должен сохранить Responses-native архитектуру ядра, но уметь работать с LLMOps / LiteLLM / OpenAI-compatible провайдерами, которые поддерживают `/v1/chat/completions`.

Главная архитектурная идея:

```text
Agent loop знает только canonical Responses-style модель:
- ResponseItem
- ResponseEvent
- tool calls
- tool outputs
- completed event

Chat Completions НЕ должен становиться отдельной веткой agent loop.
Chat Completions должен быть compatibility adapter, который притворяется Responses API.
```

## Что уже есть

В проекте уже есть:

- `WireApi::Responses`
- `WireApi::ChatCompletions`
- built-in provider `llmops`
- env-переключатель `LLMOPS_WIRE_API`
- `ChatCompletionsClient`
- базовый парсинг:
  - `content`
  - `tool_calls`
  - legacy `function_call`
  - `usage`

Ключевые файлы-кандидаты:

```text
codex-rs/model-provider-info/src/lib.rs
codex-rs/core/src/client.rs
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

## Главный принцип

Нельзя строить отдельный agent loop для Chat Completions.

Нужно сделать mapping:

```text
Responses-style Prompt / ResponseItem[]
        ↓
Chat Completions messages[]
        ↓
Chat response content/tool_calls
        ↓
ResponseEvent / ResponseItem
        ↓
существующий agent loop
```

---

# Required implementation

## 1. Найти текущую точку выбора wire_api

Найди в `codex-rs/core/src/client.rs`, где создаётся/стримится запрос к модели.

Нужно определить место, где сейчас используется:

```rust
provider.info().wire_api
```

или где по факту всегда идёт Responses path.

Добавь явную развилку:

```rust
match self.state.provider.info().wire_api {
    WireApi::Responses => {
        // existing Responses path
    }
    WireApi::ChatCompletions => {
        // new compatibility path
    }
}
```

Важно:

- Responses path не ломать.
- WebSocket path должен использоваться только для `WireApi::Responses`.
- Для `WireApi::ChatCompletions` использовать HTTP unary request.
- Если Chat Completions пока не поддерживает streaming, внутри adapter можно эмулировать stream через `mpsc::channel`, отправляя события последовательно.

---

## 2. Сделать ChatCompletions compatibility adapter

Создать отдельный модуль, например:

```text
codex-rs/core/src/chat_completions_adapter.rs
```

или рядом с client logic, если проектная структура требует иначе.

Примерная ответственность модуля:

```rust
pub(crate) async fn stream_chat_completions_as_responses(
    client: &ModelClient,
    prompt: &Prompt,
    model_info: &ModelInfo,
    ...
) -> Result<ResponseStream>
```

Названия можно адаптировать под текущий код.

Adapter должен:

1. построить `ChatCompletionsRequest`;
2. вызвать `ChatCompletionsClient::complete`;
3. преобразовать результат в `ResponseEvent`;
4. вернуть `ResponseStream`.

---

## 3. Mapping Prompt / ResponseItem -> Chat messages

Нужно реализовать converter:

```rust
fn responses_input_to_chat_messages(
    instructions: &str,
    input: &[ResponseItem],
) -> Vec<ChatCompletionsRequestMessage>
```

Минимальная логика:

```text
instructions -> system message
user input -> user message
assistant text -> assistant message
tool call output -> tool/result message, если текущая модель common.rs это поддерживает
```

Если текущий `ChatCompletionsRequestMessage` слишком простой:

```rust
pub struct ChatCompletionsRequestMessage {
    pub role: String,
    pub content: String,
}
```

то его надо расширить, чтобы поддержать tool-result сообщения.

OpenAI-compatible формат обычно такой:

```json
{
  "role": "tool",
  "tool_call_id": "call_xxx",
  "content": "..."
}
```

Поэтому нужно аккуратно расширить структуру:

```rust
pub struct ChatCompletionsRequestMessage {
    pub role: String,
    pub content: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_call_id: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub name: Option<String>,
}
```

Если в коде есть более строгий тип сообщений — использовать существующий стиль проекта.

---

## 4. Mapping tools

Сейчас `ChatCompletionsRequest` уже содержит:

```rust
pub tools: Vec<Value>,
pub tool_choice: Option<String>,
```

Нужно переиспользовать существующую генерацию tool schema, если возможно.

Для Responses уже используется:

```rust
create_tools_json_for_responses_api(&prompt.tools)
```

Нужно проверить формат результата.

Если Responses-tools несовместимы с Chat Completions, добавить converter:

```rust
fn responses_tools_to_chat_tools(tools: Vec<Value>) -> Vec<Value>
```

Цель — получить формат:

```json
{
  "type": "function",
  "function": {
    "name": "...",
    "description": "...",
    "parameters": { ... }
  }
}
```

Важно:

- не дублировать tool definitions;
- не ломать existing Responses tools;
- добавить unit tests на формат tool schema.

---

## 5. Mapping Chat output -> ResponseEvent

`ChatCompletionsOutput` содержит:

```rust
pub id: String,
pub content: String,
pub tool_calls: Vec<ChatCompletionsToolCall>,
pub usage: Option<ChatCompletionsResponseUsage>,
```

Нужно преобразовать это в поток событий:

```rust
ResponseEvent::Created
ResponseEvent::OutputTextDelta(content)
ResponseEvent::OutputItemDone(...)
ResponseEvent::Completed { ... }
```

Для tool calls нужно создать соответствующий `ResponseItem`, который существующий agent loop уже умеет выполнять.

Нужно найти в проекте, какой variant `ResponseItem` используется для function/tool call в Responses API.

Искомые места:

```bash
rg "FunctionCall"
rg "function_call"
rg "ToolCall"
rg "ResponseItem::"
rg "OutputItemDone"
```

Затем сделать mapping:

```text
ChatCompletionsToolCall {
  id,
  name,
  arguments
}
    ↓
ResponseItem::<existing function/tool call variant>
```

Важно:

- `call_id` сохранять, если он есть;
- если `id` отсутствует, генерировать стабильный id вида `chatcmpl-call-{index}`;
- `arguments` передавать как string JSON, не ломая escaping;
- если arguments невалидный JSON, не падать, а передать как string и дать loop/tool executor обработать ошибку.

---

## 6. Mapping usage

Нужно преобразовать:

```rust
ChatCompletionsResponseUsage {
    prompt_tokens,
    completion_tokens,
    total_tokens,
}
```

в существующий `TokenUsage`, если поля совпадают.

Если полного соответствия нет — заполнить доступные поля, остальные оставить default/None.

В `Completed` event нужно передать usage:

```rust
ResponseEvent::Completed {
    response_id,
    token_usage: Some(...),
    end_turn: Some(...)
}
```

Правило:

```text
если есть tool_calls -> end_turn = Some(false)
если tool_calls нет -> end_turn = Some(true)
```

---

## 7. Reasoning downgrade

Chat Completions не поддерживает Responses reasoning нативно.

Нужно явно сделать graceful downgrade:

- не отправлять `reasoning`;
- не отправлять `text.verbosity`, если Chat API не поддерживает;
- не отправлять `include`;
- не отправлять `store`;
- не отправлять `parallel_tool_calls`, если Chat provider может не поддерживать.

Если в config/model_info задано reasoning effort / summary, adapter не должен падать.

Добавить debug/warn trace, но без шума для пользователя:

```rust
tracing::debug!("reasoning controls ignored for chat-completions wire api");
```

---

## 8. Streaming compatibility

Если Chat Completions используется в non-stream режиме, всё равно вернуть `ResponseStream`.

Примерная схема:

```rust
let (tx, rx) = mpsc::channel(16);

tokio::spawn(async move {
    tx.send(Ok(ResponseEvent::Created)).await;
    if !content.is_empty() {
        tx.send(Ok(ResponseEvent::OutputTextDelta(content.clone()))).await;
        tx.send(Ok(ResponseEvent::OutputItemDone(text_item))).await;
    }
    for tool_call in tool_calls {
        tx.send(Ok(ResponseEvent::OutputItemDone(tool_item))).await;
    }
    tx.send(Ok(ResponseEvent::Completed { ... })).await;
});

Ok(ResponseStream {
    rx_event: rx,
    upstream_request_id: None,
})
```

Лучше не делать fake token-by-token streaming. Достаточно event streaming.

---

# Tests

## Unit tests

Добавить тесты на:

### 1. Chat message mapping

```text
instructions + user input -> system + user messages
```

### 2. Tool result mapping

```text
tool output ResponseItem -> role=tool message with tool_call_id
```

### 3. Chat tool call mapping

```text
ChatCompletionsToolCall -> ResponseItem tool call
```

### 4. Usage mapping

```text
ChatCompletionsResponseUsage -> TokenUsage
```

### 5. End turn behavior

```text
no tool calls -> end_turn true
has tool calls -> end_turn false
```

---

## Integration / e2e mock tests

Добавить mock e2e тест, где fake Chat Completions endpoint возвращает:

### Case A: simple final answer

```json
{
  "id": "chatcmpl-1",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "hello"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 2,
    "total_tokens": 12
  }
}
```

Ожидание:

```text
agent получает финальный ответ
tool executor не вызывается
turn завершается
```

### Case B: tool call

```json
{
  "id": "chatcmpl-2",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "",
        "tool_calls": [
          {
            "id": "call_1",
            "type": "function",
            "function": {
              "name": "shell",
              "arguments": "{\"command\":[\"pwd\"]}"
            }
          }
        ]
      }
    }
  ]
}
```

Ожидание:

```text
agent loop видит tool call
tool executor выполняет tool
tool result добавляется в следующий model request
```

### Case C: tool call -> final answer

Mock server должен вернуть:

1. первый ответ с tool call;
2. второй ответ после tool result с финальным content.

Ожидание:

```text
agent loop полностью проходит:
model -> tool call -> tool result -> model -> final answer
```

---

# Manual validation

После реализации проверить:

```bash
cargo fmt
cargo clippy --all-targets --all-features
cargo test
```

Проверить LLMOps mode:

```bash
export LLMOPS_TOKEN="..."
export LLMOPS_WIRE_API=chat-completions
export LLMOPS_BASE_URL="https://devx-copilot.tech/v1"

cargo run -p codex -- --oss-provider llmops
```

Или текущей командой запуска Gena/Codex в проекте.

Проверить сценарии:

```text
1. простой вопрос без tools
2. задача на чтение файла
3. задача на создание файла
4. задача на изменение файла
5. задача на запуск shell command
```

---

# Acceptance criteria

Задача считается выполненной, если:

- `WireApi::Responses` работает как раньше;
- `WireApi::ChatCompletions` не падает на обычном запросе;
- Chat Completions может вернуть финальный ответ;
- Chat Completions может вернуть tool call;
- существующий agent loop выполняет tool call;
- tool result уходит обратно в модель;
- после tool result модель может вернуть final answer;
- Responses-specific reasoning/websocket logic не вызывается для Chat Completions;
- есть unit tests на mapping;
- есть хотя бы один mock integration test на full loop:
  `assistant tool_call -> tool result -> assistant final`.

---

# Important constraints

Не делать:

```text
- отдельный agent loop для Chat Completions
- отдельный tool executor для Chat Completions
- hardcode только под llmops
- ломать Responses API path
- смешивать provider config и runtime loop logic
```

Делать:

```text
- Responses-native core
- Chat Completions как adapter
- минимальный diff с upstream Codex
- отдельные маленькие функции mapping
- тесты на каждый mapping
```

---

# Desired final architecture

```text
ModelClient
  ├── Responses path
  │     ├── HTTP Responses
  │     └── Responses WebSocket
  │
  └── ChatCompletions compatibility path
        ├── ResponseItem[] -> Chat messages[]
        ├── ToolSpec[] -> chat tools[]
        ├── Chat output -> ResponseEvent
        └── Chat tool_calls -> existing tool executor
```

Главное правило:

```text
Agent loop не должен знать, что под капотом был Chat Completions.
```
