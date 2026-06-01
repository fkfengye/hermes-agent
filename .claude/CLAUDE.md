# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hermes Agent is the self-improving AI agent built by Nous Research. It features a built-in learning loop, supports multiple messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal), and runs on various terminal backends (local, Docker, SSH, Daytona, Modal, Singularity).

**Tech stack**: Python 3.11+, OpenAI-compatible API, SQLite with FTS5, prompt_toolkit for CLI TUI

## Development Environment

```bash
# Create and activate venv (Python 3.11+ required)
uv venv venv --python 3.11
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows (WSL2 recommended)

# Install with all dependencies
uv pip install -e ".[all,dev]"

# Optional: RL training submodule
git submodule update --init tinker-atropos && uv pip install -e "./tinker-atropos"

# Run tests
pytest tests/ -v              # Full suite (~3000 tests)
pytest tests/test_model_tools.py -v   # Single test file
pytest tests/ -m "not integration" -n auto   # Skip integration tests
```

## Architecture

### Core Entry Points

| File | Purpose |
|------|---------|
| `run_agent.py` | `AIAgent` class — core conversation loop, tool dispatch |
| `cli.py` | `HermesCLI` class — interactive TUI with prompt_toolkit |
| `hermes_cli/main.py` | CLI entry point — all `hermes` subcommands |
| `model_tools.py` | Tool orchestration — imports all tool modules |
| `hermes_state.py` | SQLite session store with FTS5 full-text search |

### Key Modules

```
agent/               # Agent internals
├── prompt_builder.py    # System prompt assembly (identity, skills, memory)
├── context_compressor.py  # Auto-summarization at token limits
├── auxiliary_client.py    # Auxiliary LLM clients (vision, summarization)
└── display.py           # KawaiiSpinner, tool formatting

hermes_cli/          # CLI commands
├── main.py             # Entry point + argument parsing
├── config.py           # DEFAULT_CONFIG, OPTIONAL_ENV_VARS, migration
├── commands.py         # Central COMMAND_REGISTRY (CommandDef objects)
└── skin_engine.py      # Data-driven CLI theming

tools/               # Self-registering tools
├── registry.py         # Central registry (schemas, handlers, dispatch)
├── terminal_tool.py    # Terminal orchestration + backends
├── file_operations.py  # read/write/search/patch files
└── environments/       # Terminal backends (local, docker, ssh, modal, daytona)

gateway/             # Messaging platform gateway
├── run.py             # GatewayRunner — lifecycle, routing, cron
└── platforms/         # Adapters: telegram, discord, slack, whatsapp, signal
```

### Tool Discovery Flow

```
tools/registry.py  (no deps — imported by all tool files)
        ↑
tools/*.py  (each calls registry.register() at import time)
        ↑
model_tools.py  (_discover_tools() imports all tool modules)
        ↑
run_agent.py, cli.py, batch_runner.py
```

### Profiles (Multi-Instance)

`_apply_profile_override()` in `hermes_cli/main.py` sets `HERMES_HOME` before module imports. All paths must use `get_hermes_home()` from `hermes_constants` — never hardcode `~/.hermes`.

## Common Commands

```bash
hermes              # Interactive CLI
hermes model        # Choose LLM provider and model
hermes tools        # Configure enabled toolsets
hermes config set   # Set config values
hermes gateway      # Start messaging gateway
hermes setup        # Full setup wizard
hermes doctor       # Diagnostics
hermes update       # Update to latest version

# Development
pytest tests/ -v                    # Full test suite
pytest tests/gateway/ -v            # Gateway tests only
pytest tests/tools/ -v              # Tool tests only
```

## Adding New Features

### Adding a Tool (3 files)

1. Create `tools/your_tool.py` with self-registration:
   ```python
   from tools.registry import registry
   registry.register(name="my_tool", toolset="my_set", schema={...}, handler=...)
   ```
2. Add import in `model_tools.py` `_modules` list
3. Add to `toolsets.py` (either `_HERMES_CORE_TOOLS` or a new toolset)

### Adding a Slash Command (3 files)

1. Add `CommandDef` to `COMMAND_REGISTRY` in `hermes_cli/commands.py`
2. Add handler in `HermesCLI.process_command()` in `cli.py`
3. Add gateway handler in `gateway/run.py` if applicable

### Adding a Skill

Create `skills/<category>/<skill>/SKILL.md` with frontmatter (`name`, `description`, `platforms`, `required_environment_variables`). Skills are self-contained instructions — prefer skills over tools for capabilities expressible as shell commands.

## Important Policies

- **Prompt caching must not break**: Never alter past context mid-conversation, change toolsets mid-conversation, or reload memories mid-conversation. The ONLY exception is during context compression.
- **Path safety**: Use `get_hermes_home()` for all config/state paths. Use `display_hermes_home()` for user-facing messages. Never hardcode `~/.hermes`.
- **Cross-platform**: `termios`/`fcntl` are Unix-only — always catch `ImportError` and `NotImplementedError`.
- **Tool schemas**: Do not cross-reference other tools by name in schema descriptions — use dynamic resolution in `get_tool_definitions()` instead.

## Configuration

- `~/.hermes/config.yaml` — Settings (model, terminal, toolsets, compression, display)
- `~/.hermes/.env` — API keys and secrets
- Config loaders: `load_cli_config()` in `cli.py` (CLI mode), `load_config()` in `hermes_cli/config.py` (`hermes tools`, `hermes setup`), direct YAML in `gateway/run.py` (Gateway)

## User Config Location

`~/.hermes/` — contains `config.yaml`, `.env`, `auth.json`, `skills/`, `memories/`, `state.db`, `sessions/`, `cron/`
