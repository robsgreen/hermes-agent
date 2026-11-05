# Workspace: hermes-agent

Default branch: main   Remote: robsgreen/hermes-agent (fork; upstream package.json points at NousResearch/Hermes-Agent)   GitHub: yes

Notes:
- `refs/remotes/origin/HEAD` is not set locally; `git ls-remote --symref origin HEAD` reports `refs/heads/main`. Use `origin/main` as the base.
- `gh` CLI is NOT authenticated for GraphQL in this cloud session (`gh repo view` → HTTP 403 "GraphQL is not available"); GitHub access goes through the orchestrator's GitHub MCP tools (REST `gh api` may work).
- Main checkout is currently on branch `claude/hermes-discord-scheduled-reports-gxts3u`.

## Capabilities

| Capability | Status |
|------------|--------|
| Install | ✅ Python: `source ./activate` (PM provisions/syncs from `pm/lock.json` + `uv.lock`; never raw pip/uv). Test env: `python -m pm.build_env --source . --out .venv --group dev --group test` (needs Python 3.14; output dir must not exist). JS: `npm ci` (root workspaces); website: `npm ci --prefix website` |
| Build | ✅ JS only: `npm run build --workspace <ui-tui\|web\|apps/desktop>`; website `npm run build:fast --prefix website`. Python has no build step |
| Type-check | ✅ JS/TS: `npm run typecheck` per workspace (run by each workspace `check`). Python: `ty check` runs in CI as **advisory only** (`lint.yml`, `--exit-zero` diff vs base) — not a gate |
| Lint/Format | ✅ `python scripts/check` (every blocking CI lint check: ruff, code-health ratchet, footguns, shebangs, no-tmp, detached-spawns, config readers/writers, OS-marker fakes, AGENTS.md size, case-collisions, lazy-deps, profile-archives, blob-size; `--only <list>`, `--base REV`). JS: `node .github/scripts/run-workspace-checks.mjs` (every workspace `check`), or `npm run check --prefix <workspace>` |
| Unit Tests | ✅ `scripts/run_tests.sh [paths...] [pytest args]` (NEVER bare pytest; per-file subprocess isolation via `scripts/run_tests_parallel.py`, isolates HERMES_HOME, clears creds). JS: `npm run test --prefix <ui-tui\|web\|apps/desktop\|tests-js>` (vitest) |
| Integration Tests | ✅ `scripts/run_tests.sh --include-integration <files>` for `tests/e2e/**` (see below; heavy, needs built TUI, tmux, ripgrep, WAL-capable SQLite) |
| Local Dev | ✅ `source ./activate` then `hermes ...` with an isolated `HERMES_HOME`; `scripts/run-in-hermes-env CMD...` for one-off commands; JS `npm run dev --prefix <ui-tui\|web\|apps/desktop>` |
| Logging | ✅ stdlib `logging` via `hermes_logging.py` (`logging.getLogger(__name__)`, RedactingFormatter) |
| Metrics | ✅ opt-in "shared metrics" telemetry in `hermes_cli/observability/` (consent-gated) — no general metrics library |

## Modules

Python is one package rooted at `.` (pyproject `hermes-agent`, entry point `hermes = hermes_cli.main:main`); its areas are documented by per-area `AGENTS.md` files (see Docs Layout). JS is an npm workspace (root `package.json` `workspaces`).

- `.` — Hermes Agent Python core: agent loop, CLI, gateway, tools, plugins, cron, PM  (build: none, lint: `python scripts/check`, test: `scripts/run_tests.sh`)
- `apps/shared` — `@hermes/shared` shared TS code for desktop/TUI/web  (check: `npm run check --prefix apps/shared`, typecheck+lint, no tests)
- `apps/desktop` — Electron desktop shell  (build: `npm run build --prefix apps/desktop`, test: `npm run test --prefix apps/desktop`, check: `npm run check --prefix apps/desktop`)
- `apps/bootstrap-installer` — Tauri installer UI (+ Rust `src-tauri`)  (build: `npm run build --prefix apps/bootstrap-installer`, check: `npm run check --prefix apps/bootstrap-installer`; Rust: `cargo test --lib` in `apps/bootstrap-installer/src-tauri`)
- `ui-tui` — Ink terminal UI (`hermes-tui`)  (build: `npm run build --prefix ui-tui`, test: `npm run test --prefix ui-tui`, check: `npm run check --prefix ui-tui`)
- `ui-tui/packages/hermes-ink` — `@hermes/ink` forked Ink renderer  (build: `npm run build --prefix ui-tui/packages/hermes-ink`, check: typecheck)
- `web` — dashboard SPA  (build: `npm run build --prefix web`, test: `npm run test --prefix web`, check: `npm run check --prefix web`)
- `tests-js` — `@hermes/root-tests` root-level JS tests  (test: `npm run test --prefix tests-js`, check: `npm run check --prefix tests-js`)
- `website` — Docusaurus docs (not in root workspaces)  (install: `npm ci --prefix website`, build: `npm run build:fast --prefix website`)

## Module Dependencies

```
apps/desktop → apps/shared
ui-tui → apps/shared
ui-tui → ui-tui/packages/hermes-ink
web → apps/shared
. (Python: tui_gateway, e2e tests) → ui-tui (built TUI is driven by `hermes --tui`; web dashboard embeds the TUI)
```

## Docs Layout

- Root `AGENTS.md` (hub + routing table, capped at 12k chars by `python scripts/check`; root-to-area chain ≤ 30k). No `CLAUDE.md`.
- Per-area `AGENTS.md`: `agent/`, `hermes_cli/`, `gateway/`, `tools/`, `plugins/`, `tui_gateway/`, `web/`, `apps/desktop/`, `apps/desktop/src/`, `skills/`, `cron/`, `tests/`, `pm/`, `hermes_platform/`; plus `gateway/platforms/ADDING_A_PLATFORM.md`, `plugin-catalog/README.md`.
- Living long-form system docs: `website/docs/developer-guide/` (≈60 files, incl. `architecture.md`, `contributing.md`, per-subsystem internals). This is the repo's equivalent of `docs/system/`.
- User/reference docs: `website/docs/` (e.g. `website/docs/reference/package-management.md`).
- No `docs/` directory, no `docs/specs/` retro location. Contribution/workflow rules: `CONTRIBUTING.md`, `SECURITY.md`, `hermes-agent-dev` skill.
- Doc-move rule: moving a symbol means fixing `website/docs`, `skills/`, and every `AGENTS.md` in the same PR.

## Integration Tests

- Pytest markers: `integration` (external services/API keys) and `live` are excluded by default (`pyproject.toml` addopts `-m 'not integration and not live'`); `scripts/run_tests_parallel.py` skips `integration/`, `e2e/`, `docker/` dirs unless `--include-integration`.
- E2E (CI `tests.yml`): `mapfile -t files < <(find tests/e2e -name 'test_*.py' -not -path 'tests/e2e/core/upgrade/*' | sort); scripts/run_tests.sh --include-integration "${files[@]}"` with `HERMES_TEST_WORKERS=3 HERMES_TEST_FILE_TIMEOUT=900 HERMES_TEST_FILE_RETRIES=0 HERMES_E2E_REQUIRE_TUI=1`. Prereqs: PM env with extras incl. `messaging`, built TUI (`npm ci --workspace ui-tui && npm run build --workspace ui-tui`), ripgrep, tmux, WAL-capable SQLite. Runnable locally only once the 3.14 test env is built; slow.
- Upgrade E2E (`tests/e2e/core/upgrade`, separate CI job), messaging-gated gateway cases, install/desktop E2E, live-provider tests (`live-providers.yml`, need real API keys), remote backends — present but not runnable locally by the agent (secrets / disposable hosts).
- Desktop E2E: `e2e-desktop*.yml` (Electron, Xvfb) — CI only.

## Local Development

- `source ./activate` (fish/PowerShell variants) provisions the PM environment and defines `hermes` for this worktree; pick an isolated `HERMES_HOME`/`HERMES_RUNTIME_DIR` first (`website/docs/reference/package-management.md#developer-workflow`). `deactivate` restores the shell.
- Requires Python 3.14 (`>=3.14,<3.15` per CONTRIBUTING; PM provides it) and Node >= 26. The current container's `python3` is 3.13 and no `.venv` exists, so the test env must be built before `scripts/run_tests.sh` will run.
- `python scripts/check --install-hook pre-push` to run the lint lane on push.
- `docker-compose.yml` exists for containerised runs (not needed for dev).

## Observability
### Logging

stdlib `logging`, configured centrally by `hermes_logging.setup_logging()` (idempotent): rotating `agent.log`, `errors.log`, `gateway.log` (`mode="gateway"`), `gui.log` (`mode="gui"`) under `get_hermes_home()/logs`, all via one async queue and `RedactingFormatter` (secrets never reach disk). Modules use `logger = logging.getLogger(__name__)`. Ratchet rule: a broad catch that logs must keep the traceback (`logger.exception` or `exc_info=True`). Users read logs with `hermes logs`.

### Metrics

No general metrics library (no Prometheus/OpenTelemetry/statsd). First-party opt-in "shared metrics" telemetry lives in `hermes_cli/observability/` (`shared_metrics*.py`, consent via `telemetry.shared_metrics.enabled/.send` in config.yaml, per profile); lifecycle events dispatch through `hermes_cli.observability.observe_lifecycle`. Project rule: no outbound telemetry without an opt-in gate. Usage/cost accounting lives in `hermes_state_usage.py`.
