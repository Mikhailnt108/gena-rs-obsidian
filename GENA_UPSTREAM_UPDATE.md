# Gena Upstream Update

Version: v2
Applies from: `gena 0.128.0`
Replaces: `GENA_UPSTREAM_DEBUG_TEST_GATE.md`

Практичный алгоритм обновления `gena-rs` на новый upstream `openai/codex` Rust release: подготовка, merge, конфликт-фиксинг, Gena-specific баги, тесты, commit/push, debug install, smoke, release package и installer.

Codex upstream tests обязательны, но недостаточны: Gena имеет отдельные риски в branding, LLMOps provider, Chat Completions adapter, sidecar token path, installer и debug/release distribution.

## Главный Инвариант

Release package и installer нельзя считать готовыми, пока не пройдены:

1. Upstream merge завершён без конфликтов.
2. Workspace version и lock/schema files согласованы.
3. Codex mandatory checks пройдены для затронутых crates.
4. Gena bug gates пройдены.
5. Debug binary установлен и проверен.
6. Real LLMOps smoke пройден.
7. Manual TUI smoke пройден.
8. Release artifacts собраны из текущего commit и проверены.
9. Obsidian `NOW.md`, `WORKLOG.md`, [[GENA_BUGS]] обновлены.

## Быстрый Статус Перед Началом

В code repo:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project
git status --short --branch
git log --oneline -5
git remote -v
git tag --list 'rust-v*' | tail
df -h / /System/Volumes/Data codex-rs/target
```

В Obsidian:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian
git status --short --branch
sed -n '1,140p' NOW.md
```

Если code repo грязный:

- не начинать upstream merge;
- сначала понять, это пользовательские изменения, текущий hotfix или артефакты;
- не делать `git reset --hard`;
- если изменения относятся к текущей задаче, закоммитить/застейшить осознанно.

Если свободного места меньше 5GB:

- не начинать release build;
- сначала чистить старые heavy artifacts;
- `cargo clean` допустим, но помнить, что последующие tests/builds станут долгими.

## Выбор Upstream Базы

Предпочтительно обновляться по release tag:

```bash
git fetch upstream --tags
git show --no-patch --decorate rust-v0.128.0
```

Почему tag, а не `upstream/main`:

- меньше случайного API drift;
- проще фиксировать, какая версия вошла в Gena;
- легче документировать installer/package version.

## Merge Алгоритм

Обычный путь:

```bash
git switch main
git pull --ff-only origin main
git merge rust-v0.128.0
```

Если merge большой или рискованный, использовать отдельную ветку:

```bash
git switch -c chore/upstream-rust-v0.128.0
git merge rust-v0.128.0
```

После merge:

```bash
git status --short --branch
git diff --name-only --diff-filter=U
```

Merge нельзя считать законченным, пока `git diff --name-only --diff-filter=U` не пустой.

## Фикс Конфликтов

### Общий принцип

- Upstream-owned code держать максимально близко к upstream.
- Gena-specific behavior чинить в Gena-owned layers, если возможно:
  - `gena-runtime`
  - `gena-branding`
  - `gena-config`
  - `gena-upstream-adapter`
  - packaging scripts
- Если Gena fix всё же нужен в upstream-shaped crate (`codex-core`, `codex-cli`, `codex-tui`), делать минимальный targeted diff и покрывать тестом.

### Проверить после конфликтов

Обязательные quick checks:

```bash
git diff --name-only --diff-filter=U
rg -n '<<<<<<<|=======|>>>>>>>' .
```

Проверить Gena-sensitive зоны:

```bash
rg -n 'gena|llmops|LLMOPS|ProductBrand|CODEX_HOME|GENA_CODEX_HOME' codex-rs/cli codex-rs/tui codex-rs/core codex-rs/gena-runtime codex-rs/gena-upstream-adapter codex-rs/model-provider codex-rs/models-manager
```

Смотреть особенно:

- Gena binary targets (`gena`, `gena-tui`) не потерялись.
- `ProductBrand::detect_current()` всё ещё видит `gena` / `gena-tui`.
- default provider для `Gena` остаётся `llmops`.
- sidecar token path `CODEX_HOME/provider_tokens/LLMOPS_TOKEN` не сломан.
- Chat Completions adapter не регрессировал event order и system message order.
- package scripts не потеряли `GENA_CODEX_HOME` / wrapper behavior.

## Dependency / Schema / Generated Files

Если изменились `Cargo.toml` или `Cargo.lock`:

```bash
just bazel-lock-update
just bazel-lock-check
```

Если изменились `ConfigToml` или nested config types:

```bash
cd codex-rs
just write-config-schema
```

Если изменились app-server protocol/API shapes:

```bash
cd codex-rs
just write-app-server-schema
cargo test -p codex-app-server-protocol
```

## Codex Mandatory Checks

В `codex-rs`:

```bash
just fmt
git diff --check
```

Targeted tests по затронутым crates:

```bash
cargo test -p <crate>
```

Перед финализацией:

```bash
just fix -p <crate>
```

Примечания:

- Для `codex-tui` full local run может требовать:
  - `RUST_MIN_STACK=16777216 cargo test -p codex-tui`
- Не запускать полный `cargo test`/`just test` без явного решения: дорого по времени и диску.
- После `just fix` / `just fmt` не перезапускать тесты автоматически, если код не менялся вручную после этого.

## Gena Bug Gates

Эти проверки обязательны после upstream merge и после Gena hotfix, если затронуты model/provider/chat/bootstrap/TUI/branding paths.

### Branding Gate

Проверяет, что Gena entrypoints не выглядят как upstream Codex.

Tests:

```bash
cargo test -p codex-cli top_level_
cargo test -p codex-tui top_level_
```

Live checks после debug build:

```bash
target/debug/gena --version
target/debug/gena-tui --version
target/debug/gena --help | sed -n '1,28p'
target/debug/gena plugin --help | sed -n '1,12p'
target/debug/gena-tui --help | sed -n '1,10p'
```

Expected:

- `gena --version` -> `gena <current-version>`
- `gena-tui --version` -> `gena-tui <current-version>`
- `gena --help` starts with `Gena CLI`
- `gena --help` contains `Usage: gena [OPTIONS] [PROMPT]`
- `gena plugin --help` contains `Usage: gena plugin [OPTIONS] <COMMAND>`
- `gena-tui --help` contains `Usage: gena-tui [OPTIONS] [PROMPT]`

Release blockers:

- `gena --version` prints `codex-cli`.
- `gena-tui --version` prints `codex-tui`.
- top-level Gena help shows `Codex CLI`.
- top-level Gena usage shows `codex`.
- installer docs recommend `gena-tui` as primary command instead of `gena`.

### Models Catalog Gate

Tests:

```bash
cargo test -p gena-runtime gena_model_list_keeps_full_llmops_catalog_when_current_model_is_configured
cargo test -p codex-api parses_openai_compatible_models_response
cargo test -p codex-api parses_models_response
```

Must cover:

- LLMOps response shape `{"object":"list","data":[...]}`.
- `id` -> model slug.
- `modelName` -> display name.
- current configured model does not evict full remote catalog.
- sidecar token path is still supported.

### Provider Bootstrap Gate

Tests:

```bash
cargo test -p gena-runtime startup_model_tests
cargo test -p gena-runtime
```

Must cover:

- startup uses actual Gena provider, not fallback `openai`;
- `CODEX_HOME` created if missing;
- sidecar token read works where expected;
- first-run token prompt path still works;
- model cache refresh does not hide LLMOps models.

### Chat Completions Gate

Tests:

```bash
cargo test -p codex-core chat_completions_request_merges_instruction_messages_into_first_system_message
cargo test -p codex-core chat_completion_text_stream_adds_item_before_text_delta
cargo test -p codex-core chat_completion_retries_action_preamble_without_tool_call
cargo test -p codex-core chat_completion_detects_action_preamble_without_tool_call
cargo test -p codex-api endpoint::chat_completions::tests
```

Must cover:

- all `system` / `developer` instructions merged into first `system` message;
- no developer/system message appears mid-payload after model switch;
- stream event order is valid:
  - `Created`
  - `OutputItemAdded`
  - `OutputTextDelta`
  - `OutputItemDone`
  - `Completed`
- action preamble without tool call is not treated as final turn.

### TUI Gate

Tests:

```bash
cargo test -p codex-tui store_prompted_provider_token --lib
```

If `/model` UI changed:

```bash
cargo test -p codex-tui model
cargo insta pending-snapshots --manifest-path tui/Cargo.toml
```

Must cover:

- missing token prompt;
- empty token rejection;
- token persistence in sidecar;
- `/model` picker sees full catalog and current model;
- model selection does not break next turn.

## Debug Build And Install

Build:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project/codex-rs
cargo build -p codex-cli --bin gena -p codex-tui --bin gena-tui -j1
```

Verify local debug:

```bash
target/debug/gena --version
target/debug/gena-tui --version
```

Install debug explicitly:

```bash
rm -f /opt/homebrew/bin/gena-debug.bin /opt/homebrew/bin/gena.bin
cp target/debug/gena /opt/homebrew/bin/gena-debug.bin
ln /opt/homebrew/bin/gena-debug.bin /opt/homebrew/bin/gena.bin
chmod 0755 /opt/homebrew/bin/gena-debug.bin /opt/homebrew/bin/gena.bin
```

Wrapper `/opt/homebrew/bin/gena-debug` should:

- set `CODEX_HOME="${GENA_CODEX_HOME:-${HOME}/.gena-codex}"` if empty;
- `mkdir -p "$CODEX_HOME"`;
- print that debug binary is running;
- exec `/opt/homebrew/bin/gena-debug.bin`.

Verify installed debug:

```bash
gena-debug --version
stat -f '%Sm %N' /opt/homebrew/bin/gena-debug.bin /opt/homebrew/bin/gena.bin
```

## Real LLMOps Smoke

Do not print tokens.

Catalog:

```bash
token="$(tr -d '\n\r' < ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)"
curl -sS -D /tmp/gena-llmops-headers.txt -o /tmp/gena-llmops-models.json \
  -H "Authorization: Bearer ${token}" \
  -H "X-Copilot-User-Token: ${token}" \
  -H "X-Copilot-User-Agent: devxcopilot/2.0" \
  https://devx-copilot.tech/v1/models
```

Check:

- HTTP 200.
- body has `object: "list"`.
- body has non-empty `data[]`.

Non-interactive chat:

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

Check:

- response contains `OK`;
- no `System message must be at the beginning`;
- no `OutputTextDelta without active item`;
- no panic/backtrace.

Tool loop:

```bash
LLMOPS_TOKEN="$(tr -d '\n\r' < ~/.gena-codex/provider_tokens/LLMOPS_TOKEN)" \
gena-debug exec --oss --local-provider llmops \
  -m qwen3.5-35b-a3b \
  --sandbox read-only \
  --skip-git-repo-check \
  --color never \
  --json \
  --ephemeral \
  'Проверь /Users/mntabunkov/sandbox и, если там мало данных, продолжи следующей безопасной командой. Не завершай ход после фразы о следующей проверке.'
```

Check:

- after action preamble there is a command execution or explicit final answer;
- no immediate `task_complete` after a line ending with `:`.

## Manual TUI Smoke

Run:

```bash
gena-debug
```

Check header:

- provider is `llmops`;
- model is current configured model;
- no warning about missing `CODEX_HOME`.

Flow:

1. Run `/model`.
2. Confirm list contains multiple LLMOps models, not just current.
3. Select a LLMOps model.
4. Send `привет`.
5. Confirm:
   - no 400;
   - no panic/backtrace;
   - no infinite `Working`;
   - response renders as normal agent message.

## Commit And Push

For code repo:

```bash
git status --short --branch
git diff --check
git add <files>
git commit -m "<message>"
git push origin <branch>
```

For upstream merge commit on `main`, use a clear message:

```bash
git commit -m "Merge tag 'rust-v0.128.0'"
```

For post-merge hotfixes, use separate commits:

```bash
git commit -m "fix(gena): <specific bug>"
```

Do not mix:

- upstream merge;
- Gena bugfix;
- Obsidian docs;
- release artifacts.

Keep those as separate commits unless there is a hard reason not to.

## Release Package And Installer

Only after debug green path:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project/codex-rs
CARGO_BUILD_JOBS=4 scripts/package-gena-linux-macos.sh release
```

Expected macOS arm64 outputs:

- `dist/gena-v<version>-macos-arm64/`
- `dist/gena-v<version>-macos-arm64.tar.gz`
- `dist/gena-v<version>-macos-arm64.tar.gz.sha256`

If self-extract installer is needed, build it from the fresh tarball using the repo script currently used for installer generation.

Verify release artifacts:

```bash
dist/gena-v<version>-macos-arm64/gena --version
dist/gena-v<version>-macos-arm64/gena-tui --version
shasum -a 256 dist/gena-v<version>-macos-arm64.tar.gz
cat dist/gena-v<version>-macos-arm64.tar.gz.sha256
```

Expected:

- `gena <version>`
- `gena-tui <version>`
- checksum file matches tarball.

Install release only after this, then run minimal release smoke:

```bash
gena --version
gena --help | sed -n '1,20p'
```

Manual TUI release smoke:

- fresh or controlled `CODEX_HOME`;
- `/model` full LLMOps catalog;
- one prompt after model switch.

## Obsidian Sync

After every upstream/update cycle update:

- [[NOW]]
  - current branch, commits, artifact status, open blockers.
- [[WORKLOG]]
  - chronological summary, commands/checks, commits.
- [[GENA_BUGS]]
  - every discovered bug with status/version/fix/verification.
- This document if the algorithm changed.

Then commit/push Obsidian separately:

```bash
cd /Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian
git status --short --branch
git add NOW.md WORKLOG.md GENA_BUGS.md GENA_UPSTREAM_UPDATE.md
git commit -m "docs: update gena upstream status"
git push origin main
```

## Known Release Blockers To Track

These are current examples and must be reflected in [[GENA_BUGS]]:

- `gena-debug exec` requires env `LLMOPS_TOKEN` even when sidecar token exists.
- nonfatal `failed to record rollout items: thread ... not found`.
- installer artifacts lag current workspace version.
- Gena branding leaks through clap package metadata after upstream changes.

## Definition Of Done

Upstream update is done only when:

- code repo clean and pushed;
- Obsidian clean and pushed;
- upstream merge commit exists;
- Gena hotfix commits, if any, are separate and pushed;
- debug binaries report current Gena version;
- Gena bug gates pass;
- real LLMOps smoke passes;
- manual TUI smoke passes;
- release artifacts/installer are either explicitly built and verified or explicitly marked not ready in [[NOW]].
