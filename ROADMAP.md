# ROADMAP

Источник: `TASKS/CODEX_CHAT_COMPLETIONS_COMPAT_ADAPTER.md`

Цель: реализовать Chat Completions compatibility adapter для Responses-native agent loop в Gena CLI без отдельного agent loop и без поломки `WireApi::Responses`.

## Phase 0 — Baseline Discovery

1. Найти текущую точку выбора `wire_api` в `codex-rs/core/src/client.rs`.
2. Зафиксировать текущий Responses request/stream path, включая HTTP Responses и WebSocket.
3. Найти существующие типы и variants для:
   - `ResponseItem` function/tool call;
   - `ResponseEvent::OutputItemDone`;
   - `TokenUsage`;
   - `ResponseStream`.
4. Проверить текущие структуры Chat Completions в:
   - `codex-rs/codex-api/src/common.rs`;
   - `codex-rs/codex-api/src/endpoint/chat_completions.rs`.
5. Найти существующую генерацию tool schema для Responses API и оценить совместимость с Chat Completions.

## Phase 1 — Wire API Routing

1. Добавить явную развилку по `self.state.provider.info().wire_api`.
2. Оставить существующий path для `WireApi::Responses` без изменения поведения.
3. Ограничить WebSocket path только `WireApi::Responses`.
4. Для `WireApi::ChatCompletions` направить выполнение в новый compatibility adapter через HTTP unary request.
5. Добавить regression coverage, подтверждающую, что Responses path не ушёл в Chat Completions branch.

## Phase 2 — Adapter Skeleton

1. Создать отдельный модуль adapter, например `codex-rs/core/src/chat_completions_adapter.rs`.
2. Реализовать entrypoint уровня:
   - построить Chat Completions request;
   - вызвать `ChatCompletionsClient::complete`;
   - преобразовать результат в Responses-style events;
   - вернуть `ResponseStream`.
3. Для non-stream Chat Completions эмулировать event stream через `mpsc::channel`.
4. Не делать fake token-by-token streaming; отправлять последовательные semantic events.

## Phase 3 — Input Mapping

1. Реализовать converter `instructions + ResponseItem[] -> ChatCompletionsRequestMessage[]`.
2. Mapping minimum:
   - instructions -> `system`;
   - user input -> `user`;
   - assistant text -> `assistant`;
   - tool output -> `tool` message with `tool_call_id`.
3. При необходимости расширить `ChatCompletionsRequestMessage` полями:
   - `tool_call_id`;
   - `name`.
4. Сохранить OpenAI-compatible формат tool-result message.
5. Добавить unit tests:
   - instructions + user input -> system + user messages;
   - tool output ResponseItem -> role `tool` with `tool_call_id`.

## Phase 4 — Tool Schema Mapping

1. Переиспользовать существующую генерацию tool schema, если формат совместим.
2. Если Responses-tools несовместимы с Chat Completions, добавить маленький converter `responses_tools_to_chat_tools`.
3. Целевой формат tool schema:
   - `type: function`;
   - `function.name`;
   - `function.description`;
   - `function.parameters`.
4. Не дублировать tool definitions и не менять Responses tool schema.
5. Добавить unit tests на Chat Completions tool schema format.

## Phase 5 — Output Mapping

1. Реализовать mapping `ChatCompletionsOutput -> ResponseEvent`:
   - `Created`;
   - `OutputTextDelta`;
   - `OutputItemDone`;
   - `Completed`.
2. Реализовать mapping `ChatCompletionsToolCall -> ResponseItem` для существующего tool executor.
3. Сохранять `call_id`, если он есть.
4. Если `id` отсутствует, генерировать стабильный id вида `chatcmpl-call-{index}`.
5. Передавать `arguments` как JSON string без поломки escaping.
6. Если `arguments` невалидный JSON, не падать в adapter; передать дальше и дать loop/tool executor обработать ошибку.
7. Добавить unit tests:
   - Chat tool call -> ResponseItem tool call;
   - invalid arguments do not crash adapter.

## Phase 6 — Usage And Turn Boundary

1. Реализовать mapping `ChatCompletionsResponseUsage -> TokenUsage`.
2. Заполнять только доступные поля, остальные оставлять default/None.
3. Передавать usage в `ResponseEvent::Completed`.
4. Правило `end_turn`:
   - no tool calls -> `Some(true)`;
   - has tool calls -> `Some(false)`.
5. Добавить unit tests:
   - usage mapping;
   - no tool calls -> end_turn true;
   - has tool calls -> end_turn false.

## Phase 7 — Reasoning Downgrade

1. Для `WireApi::ChatCompletions` не отправлять Responses-only controls:
   - `reasoning`;
   - `text.verbosity`;
   - `include`;
   - `store`;
   - `parallel_tool_calls`, если provider может не поддерживать.
2. Если в config/model_info задан reasoning effort/summary, adapter не должен падать.
3. Добавить `tracing::debug!` о graceful downgrade без пользовательского шума.
4. Добавить coverage на сценарий с заданным reasoning config в Chat Completions mode.

## Phase 8 — Mock Integration Tests

1. Добавить mock e2e test: simple final answer.
2. Добавить mock e2e test: assistant returns tool call.
3. Добавить mock e2e test full loop:
   - model returns tool call;
   - existing tool executor runs tool;
   - tool result goes into next model request;
   - model returns final answer.
4. Проверить, что Chat Completions branch не создает отдельный tool executor.
5. Проверить, что agent loop не знает о Chat Completions internals.

## Phase 9 — Validation

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

## Acceptance Criteria

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

## Constraints

1. Не делать отдельный agent loop для Chat Completions.
2. Не делать отдельный tool executor для Chat Completions.
3. Не hardcode-ить реализацию только под `llmops`.
4. Не ломать Responses API path.
5. Не смешивать provider config и runtime loop logic.
6. Держать diff минимальным относительно upstream Codex.
7. Делать mapping маленькими отдельными функциями с тестами.
