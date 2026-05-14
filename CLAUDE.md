# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Hermes Agent is a self-improving AI agent framework (Python 3.11+, MIT license). It runs as a CLI (`hermes`), a messaging gateway (25+ platforms), a web dashboard, and a terminal UI. It supports 200+ LLM providers, has built-in learning loops (memory, skill creation, skill self-improvement), scheduled automations, subagent delegation, and RL training support.

## Build / Dev / Test Commands

### Python Setup

```bash
./setup-hermes.sh                          # Quick setup
# Or manually:
uv venv .venv --python 3.11 && source .venv/bin/activate
uv pip install -e ".[all,dev]"
```

### Testing — ALWAYS use the wrapper

```bash
scripts/run_tests.sh                                  # Full suite, CI-parity
scripts/run_tests.sh tests/gateway/                   # One directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # One test
scripts/run_tests.sh -v --tb=long                     # Pass-through pytest flags
```

The wrapper enforces: `-n 4` xdist workers, `TZ=UTC`, `LANG=C.UTF-8`, `PYTHONHASHSEED=0`, all credential env vars unset. This matches CI exactly — direct `pytest` calls cause "works locally, fails in CI" drift.

### Lint

```bash
ruff check .        # PLW1514 (unspecified-encoding) — blocking in CI
ty check            # Type checking — advisory in CI
```

### TypeScript (TUI + Web Dashboard)

```bash
# TUI (ui-tui/)
cd ui-tui && npm install && npm run dev    # Watch mode
npm run build                               # Full build
npm run type-check                          # tsc --noEmit
npm test                                    # vitest

# Web dashboard (web/)
cd web && npm run dev                       # Vite dev server
npm run build                               # Production build
```

### Docker

```bash
docker build -t hermes-agent .
HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d
```

## Architecture Overview

### Entry Points

| Entry | Module | Purpose |
|-------|--------|---------|
| `hermes` CLI | `hermes_cli/main.py` | Main CLI, dispatches all subcommands |
| `hermes-agent` | `run_agent.py` | Direct agent runner |
| `hermes --tui` | `ui-tui/src/entry.tsx` + `tui_gateway/server.py` | Ink terminal UI (React ↔ Python JSON-RPC over stdio) |
| `hermes dashboard` | `hermes_cli/web_server.py` | Web dashboard (embeds real `hermes --tui` via PTY) |
| `hermes-acp` | `acp_adapter/entry.py` | Editor integration (VS Code / Zed / JetBrains) |

### Core Modules

- **`run_agent.py`** — `AIAgent` class, core conversation loop. `__init__` takes ~60 parameters. `run_conversation()` is the main loop: calls LLM, dispatches tool calls via `handle_function_call()`, repeats until done or budget exhausted.
- **`cli.py`** — `HermesCLI` class, interactive CLI orchestrator. Uses Rich for display, prompt_toolkit for input. `process_command()` dispatches slash commands.
- **`model_tools.py`** — Tool orchestration. `discover_builtin_tools()` triggers auto-discovery; `handle_function_call()` dispatches tool calls.
- **`toolsets.py`** — Toolset definitions. `_HERMES_CORE_TOOLS` is the default bundle. Tools only appear to agents if their name is in a toolset.
- **`hermes_state.py`** — `SessionDB`, SQLite session store with FTS5 search.
- **`hermes_constants.py`** — `get_hermes_home()`, `display_hermes_home()` for profile-aware paths.

### Dependency Chain

```
tools/registry.py         (no deps — imported by all tool files)
       ↑
tools/*.py                (each calls registry.register() at import time)
       ↑
model_tools.py            (imports registry + triggers tool discovery)
       ↑
run_agent.py, cli.py, batch_runner.py
```

### Key Directories

- **`agent/`** — Provider adapters (Anthropic, Bedrock, Gemini, Codex), memory management, caching, compression, context engine, curator, prompt building, display, error classification
- **`hermes_cli/`** — CLI subcommands, setup wizard, plugins loader, skin engine, config, banner, curses UI, web server, profiles, kanban, cron, model catalog
- **`tools/`** — Tool implementations, auto-discovered via `tools/registry.py`. Includes `environments/` terminal backends (local, Docker, SSH, Modal, Daytona, Singularity)
- **`gateway/`** — Messaging gateway: `run.py` + `session.py` + `platforms/` (25+ adapters: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Home Assistant, etc.)
- **`plugins/`** — Memory providers (`honcho`, `mem0`, `supermemory`, etc.), model providers (~28 backends), context engine, image gen, kanban, observability
- **`skills/`** — 25 categories of built-in skills. `optional-skills/` for heavier/niche skills
- **`cron/`** — Scheduler: `jobs.py` + `scheduler.py`
- **`tests/`** — ~17k tests across ~900 files

### Config Locations

- **User config:** `~/.hermes/config.yaml` (settings), `~/.hermes/.env` (API keys only)
- **Logs:** `~/.hermes/logs/` — `agent.log`, `errors.log`, `gateway.log`
- **Three config loaders:** `load_cli_config()` in `cli.py`, `load_config()` in `hermes_cli/config.py`, direct YAML load in `gateway/run.py` — know which one you're in

## Adding Tools

For custom/local tools, use the plugin route: create `~/.hermes/plugins/<name>/plugin.yaml` + `__init__.py`, register via `ctx.register_tool(...)`. Don't edit core.

For core tools, changes in **2 files**:

1. **Create `tools/your_tool.py`** — define handler, call `registry.register()` at module level
2. **Add to `toolsets.py`** — tool name must appear in a toolset or it won't be exposed to agents

Auto-discovery: any `tools/*.py` with a top-level `registry.register()` is imported automatically. All handlers MUST return a JSON string.

## Adding Slash Commands

1. Add `CommandDef` to `COMMAND_REGISTRY` in `hermes_cli/commands.py`
2. Add handler in `HermesCLI.process_command()` in `cli.py`
3. If gateway-available, add handler in `gateway/run.py`

All downstream consumers (CLI, gateway, Telegram menu, Slack mapping, autocomplete, help) derive from the central registry automatically.

## Adding Configuration

- **config.yaml:** Add to `DEFAULT_CONFIG` in `hermes_cli/config.py`. Only bump `_config_version` for schema migrations (renaming keys, restructuring). New keys in existing sections are auto-merged.
- **.env:** Add to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`. `.env` is for secrets only (API keys, tokens). Non-secret settings belong in `config.yaml`.

## Critical Policies

### Prompt Caching

Do NOT alter past context, change toolsets, or rebuild system prompts mid-conversation. Cache-breaking forces dramatically higher costs. Slash commands that mutate system-prompt state must default to deferred invalidation (next session), with opt-in `--now` for immediate invalidation.

### Profile-Safe Code

Always use `get_hermes_home()` from `hermes_constants` for paths. NEVER hardcode `~/.hermes` or `Path.home() / ".hermes"` — this breaks multi-profile support. Use `display_hermes_home()` for user-facing messages.

### Plugin Isolation

Plugins MUST NOT modify core files (`run_agent.py`, `cli.py`, `gateway/run.py`, `hermes_cli/main.py`). If a plugin needs a capability the framework doesn't expose, expand the generic plugin surface.

### No New In-Tree Memory Providers

The set of built-in memory providers under `plugins/memory/` is closed. New memory backends must ship as standalone plugin repos.

## Known Pitfalls

- **No `simple_term_menu`** for new interactive menus — use `hermes_cli/curses_ui.py` instead (tmux/iTerm2 rendering bugs)
- **No `\033[K` (ANSI erase-to-EOL)** in spinner/display code — leaks as literal text under prompt_toolkit. Use space-padding.
- **No cross-tool references in schema descriptions** — tools from other toolsets may be unavailable, causing hallucinated calls. Add cross-references dynamically in `get_tool_definitions()`.
- **Don't write change-detector tests** — tests that assert specific model names, config version numbers, or enumeration counts break on routine updates. Test relationships and invariants instead.
- **Tests must not write to `~/.hermes/`** — the `_isolate_hermes_home` autouse fixture redirects to a temp dir. Profile tests must also mock `Path.home()`.
- **Squash merges from stale branches** silently revert recent fixes — always ensure branch is up to date with `main` before squash-merging.

## Slash Command Flow (TUI)

1. Built-in client commands (`/help`, `/quit`, `/clear`, `/resume`, `/copy`, `/paste`) handled locally in `app.tsx`
2. Everything else → `slash.exec` (runs in persistent `_SlashWorker` subprocess) → `command.dispatch` fallback

## Gateway Message Guards

When an agent is running, messages pass through two sequential guards: (1) base adapter queues messages when session is active, (2) gateway runner intercepts control commands (`/stop`, `/new`, `/approve`, `/deny`). New commands that must reach the runner while blocked MUST bypass both guards.
