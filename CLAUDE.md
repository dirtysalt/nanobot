# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is a lightweight, open-source personal AI agent framework. It keeps the core agent loop small and readable while supporting chat channels (Telegram, Discord, Slack, Feishu, etc.), memory, MCP, skills, and a WebUI. The codebase is intentionally simple enough to study and extend.

## Development Commands

### Install
```bash
pip install -e ".[dev]"
```

### Run Tests
```bash
pytest                          # Run all tests
pytest tests/agent/test_loop_save_turn.py -v   # Run a single test
pytest tests/channels/         # Run all channel tests
```

Tests use `pytest` with `asyncio_mode = "auto"` (configured in `pyproject.toml`).

### Lint / Format
```bash
ruff check nanobot/            # Lint
ruff format nanobot/           # Format
```

Line length is 100 characters. Target Python 3.11+. Ruff rules: E, F, I, N, W (E501 ignored).

### Core Line Count
```bash
./core_agent_lines.sh          # Print line counts for core vs. peripheral code
```

## High-Level Architecture

### Core Agent Loop
The heart of the system is `nanobot/agent/loop.py` (`AgentLoop`). It receives messages from the bus, builds context via `ContextBuilder`, runs the LLM through `AgentRunner`, executes tool calls via `ToolRegistry`, and manages memory consolidation. `nanobot/agent/runner.py` handles the actual LLM conversation turns and streaming.

### Message Bus
`nanobot/bus/queue.py` (`MessageBus`) is a simple async queue pair (`inbound`, `outbound`) that decouples chat channels from the agent core. Channels push `InboundMessage` events; the agent processes them and pushes `OutboundMessage` responses.

### Providers
`nanobot/providers/factory.py` (`make_provider`) creates the LLM provider from config. The default backend is `OpenAICompatProvider`; there are also native providers for Anthropic, Azure OpenAI, Bedrock, GitHub Copilot, and OpenAI Codex. Provider specs are registered in `nanobot/providers/registry.py`.

### Channels
`nanobot/channels/manager.py` (`ChannelManager`) initializes enabled channels and routes outbound messages. Each channel (Telegram, Discord, Slack, Feishu, WhatsApp, etc.) lives in its own module in `nanobot/channels/`. Channels are discovered via `nanobot/channels/registry.py` (pkgutil scan + entry-point plugins).

### Tools
Agent tools are registered in `nanobot/agent/tools/registry.py` (`ToolRegistry`). Built-in tools live in `nanobot/agent/tools/` and include filesystem operations, shell execution, web search/fetch, file search, ask-user, cron, notebook editing, spawn (subagent), and MCP integration. Tool schemas are sorted with builtins first, then MCP tools, for cache-friendly prompts.

### Session & Memory
`nanobot/session/manager.py` (`SessionManager`) persists conversation history. Memory consolidation ("Dream") runs on a cron schedule and is implemented in `nanobot/agent/memory.py`.

### Skills
Skills are markdown-based instruction packs stored in `nanobot/skills/`. Each skill is a directory containing a `SKILL.md` with YAML frontmatter. The skill loader is in `nanobot/agent/skills.py`.

### Config
Configuration uses Pydantic models in `nanobot/config/schema.py` and is loaded from `~/.nanobot/config.json` by `nanobot/config/loader.py`. Config accepts both camelCase and snake_case keys (`Base` model with `alias_generator=to_camel`).

### WebUI
The WebUI is a separate React + TypeScript + Vite + Tailwind frontend in `webui/`. It communicates with the gateway over WebSocket. Build output goes to `nanobot/web/dist/`. Development requires:
```bash
cd webui
bun install
bun run dev        # Dev server on http://127.0.0.1:5173
bun run build      # Production build to ../nanobot/web/dist
bun run test       # Run WebUI tests
```

### WhatsApp Bridge
`bridge/` is a standalone Node.js service using Baileys. It is built separately (`npm run build`) and started alongside the gateway.

## CLI Commands

The main entry point is `nanobot` (defined in `nanobot/cli/commands.py`):

- `nanobot onboard` — Interactive setup wizard (creates `~/.nanobot/config.json`)
- `nanobot agent` — Run the agent in interactive CLI mode
- `nanobot gateway` — Start the gateway (WebSocket + REST API for channels and WebUI)
- `nanobot serve` — Start the OpenAI-compatible API server
- `nanobot status` — Show runtime status
- `nanobot provider login <name>` / `nanobot provider logout <name>` — OAuth login/logout for supported providers

## Branching Strategy

- `main` — Stable releases. Target for bug fixes and docs.
- `nightly` — Experimental features. Target for new features, refactoring, and API changes.

Stable features are cherry-picked from `nightly` into individual PRs targeting `main` approximately once a week. When in doubt, target `nightly`.

## Code Style Guidelines

- Prefer the smallest change that solves the real problem.
- Optimize for the next reader, not for cleverness.
- Keep boundaries clean; avoid unnecessary new abstractions.
- The codebase is intentionally small and readable — resist bloat.
- All async code uses `asyncio`.
- Use `from __future__ import annotations` in new files.
