# GENA BUGS

Реестр Gena-specific багов: branding, LLMOps integration, Chat Completions adapter, installer/debug pipeline и runtime поведение. Цель документа — не терять баги между upstream sync, debug gate и release packaging.

## Правила Ведения

- Каждый баг должен иметь статус: `open`, `fixed`, `needs decision`, `obsolete`.
- Указывать версию, где баг найден, и версию/commit, где он исправлен.
- Если баг найден на debug, release/installer нельзя считать готовым, пока он не отражён здесь и в [[GENA_UPSTREAM_UPDATE]] при необходимости.
- Internal upstream names (`codex-*` crate/type names) сами по себе не баг. Багом считается user-visible leak в `gena` entrypoints, installer, docs, runtime errors или UI.

## Краткая Таблица

| ID | Тема | Статус | Найдено | Исправлено |
|---|---|---:|---|---|
| GENA-BUG-001 | `gena --version` печатает `codex-cli` | fixed | `0.128.0` | `0.128.0`, `58d464d38e` |
| GENA-BUG-002 | `gena-tui --version` печатает `codex-tui` | fixed | `0.125.0`, `0.128.0` | `0.128.0`, `58d464d38e` |
| GENA-BUG-003 | `gena --help` показывает `Codex CLI` / `Usage: codex` | fixed | `0.128.0` | `0.128.0`, `58d464d38e` |
| GENA-BUG-004 | subcommand help показывает `Usage: codex ...` для Gena | fixed | `0.128.0` | `0.128.0`, `58d464d38e` |
| GENA-BUG-005 | hardcoded `Codex` в update/auth/runtime messages | fixed | `0.128.0` | `0.128.0`, `58d464d38e` |
| GENA-BUG-006 | `/model` теряет LLMOps catalog и показывает только current model | fixed | `0.120.0`/`0.125.0` | `0.125.0` cycle |
| GENA-BUG-007 | LLMOps `/models` real shape `data[]` не парсился | fixed | `0.125.0` | `0.125.0`, `2197cbef3` |
| GENA-BUG-008 | Chat Completions 400 `System message must be at the beginning` | fixed | `0.125.0` | `0.125.0`, `2c9004753` |
| GENA-BUG-009 | `OutputTextDelta without active item` | fixed | `0.125.0` | `0.125.0`, `15bbd633b` |
| GENA-BUG-010 | action preamble без tool call считается final turn | fixed | `0.125.0` | `0.125.0`, `bd164e1e1a` / `8994663bf5` |
| GENA-BUG-011 | assistant text + tool call ставил `end_turn: true` | fixed | `0.125.0` | `0.125.0` hotfix cycle |
| GENA-BUG-012 | first run без `LLMOPS_TOKEN` не спрашивал токен | fixed | `0.120.0` | `0.120.0`, `d5f73631d` |
| GENA-BUG-013 | refresh remote models не работал с env-backed provider auth | fixed | `0.120.0` | `0.120.0`, `c714af8a5` |
| GENA-BUG-014 | `gena-debug exec` требует env token при наличии sidecar token | needs decision | `0.125.0` | not fixed |
| GENA-BUG-015 | nonfatal `failed to record rollout items: thread ... not found` | open | `0.125.0` | not fixed |
| GENA-BUG-016 | release installer может вернуть старый build | fixed by process | `0.120.0` | gate/process |
| GENA-BUG-017 | `0.128.0` installer отсутствует после upstream merge | open | `0.128.0` | not built |

## Bugs

### GENA-BUG-001 — `gena --version` печатает `codex-cli`

Status: `fixed`

Found in: `0.128.0` after upstream merge.

Symptom:
- `target/debug/gena --version` возвращал `codex-cli 0.128.0`.

Cause:
- `gena` — bin target внутри Rust package `codex-cli`.
- clap default `version` использовал package metadata (`CARGO_PKG_NAME`), а не Gena command identity.

Fix:
- `codex-rs/cli/src/main.rs`
- top-level clap command строится через `ProductBrand::detect_current()`.
- Явно задаются `.name("gena")`, `.bin_name("gena")`, `.version(env!("CARGO_PKG_VERSION"))`.

Fixed in:
- `0.128.0`
- commit `58d464d38e` — `fix(gena): brand cli help and version output`

Verification:
- `target/debug/gena --version` -> `gena 0.128.0`
- `cargo test -p codex-cli top_level_`

### GENA-BUG-002 — `gena-tui --version` печатает `codex-tui`

Status: `fixed`

Found in:
- `0.125.0` release artifacts: `dist/gena-v0.125.0-macos-arm64/gena-tui --version` -> `codex-tui 0.125.0`
- `0.128.0` debug after upstream merge: `target/debug/gena-tui --version` -> `codex-tui 0.128.0`

Cause:
- `gena-tui` — bin target внутри Rust package `codex-tui`.
- clap default version name came from package metadata.

Fix:
- `codex-rs/tui/src/main.rs`
- top-level TUI clap command получает brand-specific command name:
  - `Codex` -> `codex-tui`
  - `AqaCodex` -> `aqa-codex-tui`
  - `Gena` -> `gena-tui`

Fixed in:
- `0.128.0`
- commit `58d464d38e`

Verification:
- `target/debug/gena-tui --version` -> `gena-tui 0.128.0`
- `cargo test -p codex-tui top_level_`

### GENA-BUG-003 — `gena --help` показывает `Codex CLI` / `Usage: codex`

Status: `fixed`

Found in: `0.128.0` after upstream merge.

Symptom:
- `gena --help` начинался с `Codex CLI`.
- Usage был `codex [OPTIONS] [PROMPT]`.

Cause:
- `codex-rs/cli/src/main.rs` had hardcoded clap attributes:
  - `bin_name = "codex"`
  - `override_usage = "codex ..."`
  - doc comment `Codex CLI`

Fix:
- hardcoded top-level identity removed.
- command title and long about are loaded from `gena_branding::cli_identity(ProductBrand::detect_current())`.

Fixed in:
- `0.128.0`
- commit `58d464d38e`

Verification:
- `target/debug/gena --help` -> starts with `Gena CLI`
- `target/debug/gena --help` -> contains `Usage: gena [OPTIONS] [PROMPT]`

### GENA-BUG-004 — subcommand help показывает `Usage: codex ...`

Status: `fixed`

Found in: `0.128.0`.

Symptom:
- `gena plugin --help` showed `Usage: codex plugin ...`.

Cause:
- nested `PluginCli` had hardcoded `#[command(bin_name = "codex plugin")]`.

Fix:
- removed hardcoded nested `bin_name`.
- root branded command now lets clap propagate the correct binary name into subcommand usage.

Fixed in:
- `0.128.0`
- commit `58d464d38e`

Verification:
- `target/debug/gena plugin --help` -> `Usage: gena plugin [OPTIONS] <COMMAND>`

### GENA-BUG-005 — hardcoded `Codex` в update/auth/runtime messages

Status: `fixed`

Found in: `0.128.0`.

Symptoms:
- update path printed `Updating Codex ...` / `Please restart Codex`.
- debug update error referenced `codex update`.
- missing auth error said `No Codex auth ... run codex login`.
- dumb-terminal warning said `Codex's interactive TUI`.

Fix:
- `codex-rs/cli/src/main.rs`
  - update messages use `gena_branding::update_in_progress_message` and `update_success_message`.
  - debug update error uses `current_cli_command_name()` and product display name.
  - dumb-terminal warning uses `ProductBrand::detect_current().display_name()`.
- `codex-rs/gena-runtime/src/auth.rs`
  - missing auth error now says `No auth is configured; please run <brand> login`.
- `codex-rs/gena-runtime/src/interaction.rs`
  - generic dumb-terminal message no longer hardcodes Codex.

Fixed in:
- `0.128.0`
- commit `58d464d38e`

Verification:
- `cargo test -p codex-cli top_level_`
- `cargo test -p gena-runtime`

### GENA-BUG-006 — `/model` показывает только current model вместо полного LLMOps catalog

Status: `fixed`

Found in: `0.120.0`/`0.125.0` cycles.

Symptom:
- После старта или выбора модели `/model` мог показывать только текущую модель.
- Full LLMOps catalog исчезал.

Causes:
- remote refresh/cache path не учитывал env-backed provider auth.
- startup/runtime path мог fallback-иться на wrong provider.
- LLMOps real `/models` shape also differed from mocked assumptions.

Fix:
- `codex-rs/models-manager/src/manager.rs`
- `codex-rs/gena-runtime/src/startup_model_tests.rs`
- parser/model list tests updated to preserve full LLMOps catalog.

Fixed in:
- `0.120.0`/`0.125.0` cycle

Verification:
- `cargo test -p codex-models-manager`
- `cargo test -p gena-runtime startup_model_tests`
- `cargo test -p gena-runtime gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured`

### GENA-BUG-007 — LLMOps `/models` real shape `data[]` не парсился

Status: `fixed`

Found in: `0.125.0`.

Symptom:
- Tests expected an internal Codex-like `{"models":[...]}` response.
- Real LLMOps returned OpenAI-compatible `{"object":"list","data":[...]}`.

Fix:
- `codex-api` parser supports both shapes.
- Regression test mocks real `data[]` shape.

Fixed in:
- `0.125.0`
- commit `2197cbef3` — `fix(gena): parse llmops model catalog`

Verification:
- `cargo test -p codex-api parses_openai_compatible_models_response`
- `cargo test -p codex-api parses_models_response`

### GENA-BUG-008 — Chat Completions 400 `System message must be at the beginning`

Status: `fixed`

Found in: `0.125.0`.

Symptom:
- After model switch through `/model`, LLMOps returned HTTP 400:
  - `System message must be at the beginning`

Cause:
- Chat Completions payload could contain separate developer/model-switch instruction after first system message.

Fix:
- Chat Completions request builder merges all `system` / `developer` instruction messages into one first `system` message.

Fixed in:
- `0.125.0`
- commit `2c9004753` — `fix(gena): keep chat system message first`

Verification:
- `cargo test -p codex-core chat_completions_request_merges_instruction_messages_into_first_system_message`
- real LLMOps sanity check returned `OK` without 400.

### GENA-BUG-009 — `OutputTextDelta without active item`

Status: `fixed`

Found in: `0.125.0`.

Symptom:
- TUI/non-interactive debug path panicked or errored with:
  - `OutputTextDelta without active item`

Cause:
- Chat Completions adapter emitted `OutputTextDelta` before `OutputItemAdded`.

Fix:
- adapter event order changed to:
  - `Created`
  - `OutputItemAdded`
  - `OutputTextDelta`
  - `OutputItemDone`
  - `Completed`

Fixed in:
- `0.125.0`
- commit `15bbd633b` — `fix(gena): emit chat item before text delta`

Verification:
- `cargo test -p codex-core chat_completion_text_stream_adds_item_before_text_delta`
- real LLMOps sanity check did not hit the panic.

### GENA-BUG-010 — action preamble без tool call считается final turn

Status: `fixed`

Found in: `0.125.0`.

Symptom:
- `gena-debug` turn ended after text like:
  - `Посмотрим общий список крупных файлов и папок в корне:`
- No next command execution followed.

Cause:
- Chat Completions adapter treated an action-like preamble without tool call as final assistant answer.

Fix:
- retry/continuation nudge for action preamble without tool call.
- keep chat prelude turn open before expected tool call.

Fixed in:
- `0.125.0`
- commits:
  - `bd164e1e1a` — `fix(gena): retry chat action preamble without tool call`
  - `8994663bf5` — `fix(gena): keep chat prelude turn open before tool call`

Verification:
- `cargo test -p codex-core chat_completion_retries_action_preamble_without_tool_call`
- `cargo test -p codex-core chat_completion_detects_action_preamble_without_tool_call`

### GENA-BUG-011 — assistant text + tool call ставил `end_turn: true`

Status: `fixed`

Found in: `0.125.0` hotfix cycle.

Symptom:
- Chat Completions response can contain assistant text and `tool_calls` in one response.
- Synthesized assistant message was marked `end_turn: true` even when tool call followed.
- This created a false turn boundary before tool result.

Fix:
- `codex-rs/core/src/client.rs`
- synthesized assistant message now sets `end_turn: false` when `tool_items` is not empty.

Fixed in:
- `0.125.0` hotfix cycle.

Verification:
- `chat_completion_text_before_tool_call_does_not_end_turn`
- `chat_completion_text_before_legacy_tool_call_does_not_end_turn`
- `chat_completions_text_before_tool_call_runs_tool_loop_to_completion`

### GENA-BUG-012 — first run без `LLMOPS_TOKEN` не спрашивал токен

Status: `fixed`

Found in: `0.120.0`.

Symptom:
- Fresh `CODEX_HOME`, no `LLMOPS_TOKEN` in env.
- TUI did not prompt for token.
- Startup failed with provider token error.

Fix:
- `codex-rs/tui/src/lib.rs`
- `RuntimeProviderTokenState::NeedsPrompt` now triggers interactive prompt, hidden token read, sidecar persistence, and startup continuation.

Fixed in:
- `0.120.0`
- commit `d5f73631d` — `fix(gena): prompt for llmops token on first run`

Verification:
- `cargo test -p codex-tui store_prompted_provider_token --lib`
- release onboarding smoke created `provider_tokens/LLMOPS_TOKEN`.

### GENA-BUG-013 — refresh remote models не работал с env-backed provider auth

Status: `fixed`

Found in: `0.120.0`.

Symptom:
- Old `models_cache.json` was rejected due to client version mismatch.
- Remote refresh did not run for provider auth via `env_key`.
- LLMOps models disappeared from startup picker.

Fix:
- `codex-rs/models-manager/src/manager.rs`
- refresh now works when provider token is available through env-backed auth.

Fixed in:
- `0.120.0`
- commit `c714af8a5` — `fix(gena): refresh llmops models with env auth`

Verification:
- `cargo test -p codex-models-manager`
- runtime smoke confirmed `/v1/models?client_version=0.120.0` and cache with remote model.

### GENA-BUG-014 — `gena-debug exec` требует env token при наличии sidecar token

Status: `needs decision`

Found in: `0.125.0`.

Symptom:
- Without `LLMOPS_TOKEN` env, `gena-debug exec` failed:
  - `Missing environment variable: LLMOPS_TOKEN`
- Sidecar token existed at:
  - `~/.gena-codex/provider_tokens/LLMOPS_TOKEN`

Current workaround:
- Export token explicitly:
  - `LLMOPS_TOKEN="$(tr -d '\n\r' < ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)" gena-debug exec ...`

Decision needed:
- Decide whether non-interactive `exec` should read sidecar provider tokens automatically, matching TUI startup behavior.
- If yes, add regression test and fix in provider/auth resolution path.

Fixed in: not fixed.

### GENA-BUG-015 — nonfatal `failed to record rollout items: thread ... not found`

Status: `open`

Found in: `0.125.0` real debug smoke.

Symptom:
- After successful `gena-debug exec` with LLMOps returning `OK`, nonfatal log remains:
  - `failed to record rollout items: thread ... not found`

Impact:
- Does not block basic response path.
- May indicate rollout/thread persistence mismatch in ephemeral/non-interactive path.

Fixed in: not fixed.

Next:
- Reproduce with `--json --ephemeral`.
- Determine whether this is expected for ephemeral exec or a state bug.

### GENA-BUG-016 — release installer может вернуть старый build

Status: `fixed by process`

Found in: `0.120.0`.

Symptom:
- User reinstalled from an old `gena-v0.120.0-macos-arm64-installer.sh` in Downloads.
- Old installer overwrote newer fixed binary and reintroduced old behavior.

Fix:
- Rebuilt release artifacts after fixes.
- [[GENA_UPSTREAM_UPDATE]] now explicitly blocks release/installer before debug green path.
- Obsidian `NOW.md` must record artifact versions and mtimes.

Fixed in:
- process/gate, not only code.

### GENA-BUG-017 — `0.128.0` installer отсутствует после upstream merge

Status: `open`

Found in: `0.128.0` cycle.

Symptom:
- Code `main` is at `0.128.0`.
- Debug binaries exist.
- `codex-rs/dist` still contains only `0.125.0` macOS arm64 package and installer.

Impact:
- Users can still accidentally install stale `0.125.0`.
- `0.128.0` cannot be called release-ready until package and installer are rebuilt and smoke-tested.

Next:
- Install debug `0.128.0` explicitly.
- Run real LLMOps smoke and manual TUI smoke.
- Build `gena-v0.128.0-macos-arm64.tar.gz`, `.sha256`, and installer only after debug green path.

Fixed in: not fixed.

## Release Blocking Checklist

Before any Gena release package/installer:

- [ ] No `gena --version` / `gena-tui --version` branding leak.
- [ ] `gena --help` starts with `Gena CLI`.
- [ ] `gena` is documented as primary terminal command; `gena-tui` is fallback/direct TUI.
- [ ] Full LLMOps model catalog is available.
- [ ] Chat Completions payload keeps first system message valid.
- [ ] stream event order is valid for TUI.
- [ ] action preamble without tool call does not prematurely end turn.
- [ ] debug real LLMOps smoke passes.
- [ ] manual TUI smoke passes.
- [ ] installer artifacts match current workspace version.
