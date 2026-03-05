# Migration Plan: context-mode → Generic MCP Server

## Overview

This document records the plan to transform `context-mode` from a Claude Code-specific MCP plugin into a platform-agnostic MCP server that works with:

- **OpenCode** (primary target) — open-source terminal AI coding assistant with native MCP support
- **ZeroClaw** — Rust-based terminal AI coding assistant (OpenClaw alternative) with native MCP support
- **VS Code** — via GitHub Copilot Chat MCP integration or Cursor
- **LM Studio** — local inference runner with MCP support
- **Ollama + OpenWebUI** — local inference via OpenWebUI's MCP proxy
- **Claude Code** — retained as backward-compatible (existing users not broken)

---

## Problem Statement

The original codebase was tightly coupled to Claude Code via:

1. `CLAUDE_PROJECT_DIR` environment variable — hardcoded assumption about Claude Code's project root injection → **completely removed**, replaced by `PROJECT_DIR`
2. `~/.claude/plugins/installed_plugins.json` — Claude Code marketplace plugin registry self-heal code in `start.mjs` and `hooks/pretooluse.mjs`
3. `~/.claude/settings.json` — Claude Code permission system (security policies)
4. `.claude-plugin/` — Claude Code marketplace plugin manifest format
5. `hooks/` directory — Claude Code lifecycle hooks (`PreToolUse`, `SessionStart`)
6. `package.json` description and keywords referencing "claude" and "claude-code"
7. `src/cli.ts` setup command offering only "Claude Code" or "manual" install options
8. `.mcp.json` using `${CLAUDE_PLUGIN_ROOT}` — a Claude Code plugin variable

---

## Changes Made

### 1. Source Code (`src/`)

| File | Change |
|------|--------|
| `src/server.ts` | Replace `CLAUDE_PROJECT_DIR` with `PROJECT_DIR` (completely removed) |
| `src/cli.ts` | Add OpenCode, ZeroClaw, VS Code, LM Studio, Ollama setup options to the `setup` command |

### 2. Runtime Bootstrap (`start.mjs`)

- **Removed**: Claude plugin registry self-heal code that read/wrote `~/.claude/plugins/installed_plugins.json`
- **Updated**: Use `PROJECT_DIR` env var exclusively (`CLAUDE_PROJECT_DIR` completely removed)

### 3. Configuration (`.mcp.json`)

- **Changed**: From `${CLAUDE_PLUGIN_ROOT}/start.mjs` (Claude Code plugin variable) to `npx -y context-mode` (works anywhere Node.js is available)

### 4. Package Metadata (`package.json`)

- **Updated**: Description to mention all supported platforms
- **Updated**: Keywords — removed `claude`, `claude-code`; added `opencode`, `zeroclaw`, `vscode`, `lm-studio`, `ollama`, `openwebui`

### 5. Hooks (`hooks/`)

The hooks directory is **retained but scoped to Claude Code**. These hooks intercept Claude Code's `PreToolUse` and `SessionStart` events to redirect tool calls through the MCP sandbox. They are not applicable to other platforms and are gracefully ignored by non-Claude Code clients.

### 6. Plugin Manifest (`.claude-plugin/`)

The `.claude-plugin/` directory is **retained** for Claude Code marketplace compatibility. It is ignored by all other platforms.

---

## Environment Variables

| Variable | Platform | Description |
|----------|----------|-------------|
| `PROJECT_DIR` | All platforms | Project root for security policy resolution |

---

## Security Policy Resolution

The security module reads permission policies from `settings.json` files. On non-Claude platforms these files won't exist, so the module returns empty policy arrays and all commands are permitted (fail-open). This is the correct behavior — security policies are opt-in.

---

## Platform Setup Guides

See the other files in this folder:

- [`OPENCODE-SETUP.md`](./OPENCODE-SETUP.md)
- [`ZEROCLAW-SETUP.md`](./ZEROCLAW-SETUP.md)
- [`VSCODE-SETUP.md`](./VSCODE-SETUP.md)
- [`LM-STUDIO-SETUP.md`](./LM-STUDIO-SETUP.md)
- [`OPENWEBUI-SETUP.md`](./OPENWEBUI-SETUP.md)
- [`CLAUDE-REMOVAL.md`](./CLAUDE-REMOVAL.md)
