# NOW

## Current Goal
Довести `gena` после upstream `rust-v0.128.0` до release package/installer: debug build установлен и проверен, дальше real LLMOps smoke, manual TUI smoke и сборка `0.128.0` artifacts.

## State
- Кодовый репозиторий: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-project`.
- Obsidian vault: `/Users/mntabunkov/my_github_projects/gena-rs/gena-rs-obsidian`.
- Ветка кода: `main`, синхронизирована с `origin/main`.
- Последние ключевые commits:
  - `4303eb9c24` — `Merge tag 'rust-v0.128.0'`
  - `58d464d38e` — `fix(gena): brand cli help and version output`
  - `c79ac7c704` — `fix(gena): restore tui provider token onboarding`
- Upstream update до `rust-v0.128.0` завершён и запушен.
- Rust workspace version: `0.128.0`.
- Рабочее дерево кода clean после `gena-debug` token prompt regression fix.
- Agent entrypoint consolidated: `AGENTS.md` is now the single required startup file; old `AGENT_BOOT.md` was removed.

## Built Artifacts
- Debug binaries собраны локально:
  - `codex-rs/target/debug/gena` -> `gena 0.128.0`
  - `codex-rs/target/debug/gena-tui` -> `gena-tui 0.128.0`
- Debug binaries установлены:
  - `/opt/homebrew/bin/gena-debug.bin` -> `gena 0.128.0`
  - `/opt/homebrew/bin/gena.bin` -> `gena 0.128.0`
  - `/opt/homebrew/bin/gena-tui.bin` -> `gena-tui 0.128.0`
  - active PATH nvm shims also updated:
    - `~/.nvm/versions/node/v22.22.2/bin/gena.bin` -> `gena 0.128.0`
    - `~/.nvm/versions/node/v22.22.2/bin/gena-tui.bin` -> `gena-tui 0.128.0`
- Release artifacts для `0.128.0` пока НЕ собраны.
- Installer для `0.128.0` пока НЕ готов.
- В `codex-rs/dist` всё ещё лежат старые `0.125.0` artifacts:
  - `gena-v0.125.0-macos-arm64.tar.gz`
  - `gena-v0.125.0-macos-arm64.tar.gz.sha256`
  - `gena-v0.125.0-macos-arm64-installer.sh`

## Fixed Bugs In 0.128.0 Cycle
- Branding/version leak after upstream merge:
  - `gena --version` печатал `codex-cli 0.128.0`.
  - `gena-tui --version` печатал `codex-tui 0.128.0`.
  - `gena --help` показывал `Codex CLI` and `Usage: codex ...`.
  - `gena plugin --help` показывал `Usage: codex plugin ...`.
  - часть help/update/runtime messages содержала hardcoded `Codex`.
- Fix:
  - top-level clap command теперь строится через `ProductBrand::detect_current()`.
  - `gena` получает `Gena CLI`, `Usage: gena ...`, `gena 0.128.0`.
  - `gena-tui` получает `Usage: gena-tui ...`, `gena-tui 0.128.0`.
  - update messages, debug-build update error, auth error and dumb-terminal warning больше не hardcode-ят `codex`.
- `gena-debug` TUI token prompt regression:
  - before fix: first request failed with `Missing environment variable: LLMOPS_TOKEN`.
  - restored startup call to `prepare_runtime_provider_token()` after `Config` load.
  - sidecar token path is loaded before the first request; otherwise interactive TUI prompts for token.
  - non-interactive startup now fails early with provider-specific message instead of later request failure.
  - TUI session/status headers now use current product brand and should render `Gena` for `gena-debug`.

## Checks Completed
- Для upstream merge:
  - `just fmt` PASS.
  - `git diff --check` PASS.
  - `cargo test -p codex-config -p codex-models-manager -p codex-model-provider -p gena-runtime -p gena-upstream-adapter` PASS.
  - `cargo test -p codex-core` PASS after rerun of one flaky timeouted test.
  - `RUST_MIN_STACK=16777216 cargo test -p codex-tui` PASS.
  - `cargo insta pending-snapshots --manifest-path tui/Cargo.toml` -> no pending snapshots.
  - `just write-config-schema` PASS.
  - `just bazel-lock-update` PASS.
  - `just bazel-lock-check` PASS.
  - `just fix -p codex-config -p codex-model-provider -p codex-models-manager -p codex-core -p codex-tui -p gena-runtime -p gena-upstream-adapter` PASS with existing warnings.
- Для branding fix:
  - `cargo test -p codex-cli top_level_` PASS.
  - `cargo test -p codex-tui top_level_` PASS.
  - `cargo test -p gena-runtime` PASS.
  - `just fix -p codex-cli -p codex-tui` PASS.
  - `just fmt` PASS.
  - `cargo build -p codex-cli --bin gena -p codex-tui --bin gena-tui -j1` PASS.
  - live checks:
    - `target/debug/gena --version` -> `gena 0.128.0`
    - `target/debug/gena-tui --version` -> `gena-tui 0.128.0`
    - `target/debug/gena --help` -> `Gena CLI`, `Usage: gena ...`
    - `target/debug/gena plugin --help` -> `Usage: gena plugin ...`
- Для установленной debug-сборки:
  - `gena --version` -> `gena 0.128.0`
  - `gena-tui --version` -> `gena-tui 0.128.0`
  - `gena-debug --version` -> `gena 0.128.0`
  - `gena --help` -> `Gena CLI`, `Usage: gena ...`
- Для `gena-debug` token prompt fix:
  - `cargo test -p gena-types detects_brands_from_program_stem` PASS.
  - `cargo test -p codex-tui provider_token` PASS for stored-token path.
  - `cargo test -p codex-tui prompt_for_missing_provider_env_key_loads_sidecar_token` PASS.
  - `just fix -p codex-tui -p gena-types` PASS.
  - `just fmt` PASS.
  - `git diff --check` PASS.
  - `cargo build -p codex-cli --bin gena -p codex-tui --bin gena-tui -j1` PASS.
  - installed debug binaries refreshed in `/opt/homebrew/bin` and nvm shim dir.
  - no-token/non-interactive smoke now reports provider-specific `LLMOPS_TOKEN` requirement before TUI request path.

## Open Bugs / Risks
- Release installer `0.128.0` not built yet.
- Manual TUI smoke through installed debug binary is still not confirmed for `0.128.0`.
- Pre-existing runtime issues still need separate decision:
  - `gena-debug exec` missing env when only sidecar token exists.
  - nonfatal `failed to record rollout items: thread ... not found` after successful `gena-debug exec`.
- TUI test caveat:
  - local `codex-tui` full test needs `RUST_MIN_STACK=16777216`; default stack overflows one test locally.

## Next Step
Run real LLMOps smoke and manual TUI smoke on installed `0.128.0` debug, then build `gena-v0.128.0-macos-arm64` release package/installer only after debug green path.
