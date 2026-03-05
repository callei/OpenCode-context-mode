# Claude-Specific Code Removal

## Overview

This document records what Claude Code-specific code was removed or replaced, and why, as part of making `context-mode` a platform-agnostic MCP server.

---

## Removed: Claude Plugin Registry Self-Heal (start.mjs)

### What it was

`start.mjs` contained ~50 lines of self-heal code that:
1. Scanned `~/.claude/plugins/cache/` for version directories
2. Read and updated `~/.claude/plugins/installed_plugins.json` — Claude Code's internal plugin registry
3. Bumped plugin `installPath` and `version` fields to point to the newest cache directory

### Why removed

This code directly mutates Claude Code's internal plugin state. On all non-Claude platforms this code:
- Throws errors silently (the file doesn't exist)
- Is entirely irrelevant to server startup

The self-heal was a workaround for a Claude Code marketplace caching quirk. With the generic MCP deployment model (e.g., `npx -y context-mode`), the platform's MCP client handles version management.

### Files affected

- `start.mjs` — lines 16–63 removed; file now only handles env var setup and server boot

---

## Changed: `CLAUDE_PROJECT_DIR` → `PROJECT_DIR` (completely removed)

### What it was

The server read `process.env.CLAUDE_PROJECT_DIR` to find the project root for:
- Passing to `PolyglotExecutor` as the sandbox working directory
- Loading security policies from `.claude/settings.json` in the project

### What changed

`CLAUDE_PROJECT_DIR` has been **completely removed**. All call sites now read only `process.env.PROJECT_DIR`.

### Files affected

- `src/server.ts` — lines 32, 87, 116, 144
- `hooks/pretooluse.mjs` — 5 call sites
- `start.mjs` — env var default uses `PROJECT_DIR` only

---

## Changed: `.mcp.json`

### What it was

```json
{
  "mcpServers": {
    "context-mode": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/start.mjs"]
    }
  }
}
```

`${CLAUDE_PLUGIN_ROOT}` is a variable injected by Claude Code when it loads a plugin. It resolves to the plugin's installation cache directory. This variable is undefined in all other MCP clients.

### What changed

```json
{
  "mcpServers": {
    "context-mode": {
      "command": "npx",
      "args": ["-y", "context-mode"]
    }
  }
}
```

`npx -y context-mode` works with any MCP client that can spawn subprocesses. It fetches the package from npm on first run and caches it locally.

---

## Changed: `package.json` metadata

### What changed

| Field | Before | After |
|-------|--------|-------|
| `description` | "Claude Code MCP plugin…" | "MCP server that saves context window…" |
| `keywords` | includes `claude`, `claude-code` | replaced with `opencode`, `vscode`, `lm-studio`, `ollama`, `openwebui` |

---

## Changed: `src/cli.ts` setup command

### What it was

The `setup` command offered only two options:
1. **Claude Code** (`claude mcp add …`)
2. Manual (show `.mcp.json`)

### What changed

Added platform-specific setup options:
- **OpenCode** — writes `~/.config/opencode/opencode.json`
- **VS Code** — writes `.vscode/mcp.json`
- **LM Studio** — shows LM Studio config snippet
- **Ollama + OpenWebUI** — shows OpenWebUI config snippet
- **Claude Code** — retained (backward-compatible)
- **Manual** — retained (show raw JSON)

---

## Retained: `hooks/` directory

The `hooks/` directory (`pretooluse.mjs`, `sessionstart.mjs`, `routing-block.mjs`) is **retained** because:

- These hooks work correctly for Claude Code users
- Removing them would break the Claude Code plugin use case
- On all other platforms these files are simply never executed

They are not applicable to OpenCode, VS Code, LM Studio, or OpenWebUI, which do not support the Claude Code hook protocol.

---

## Retained: `.claude-plugin/` directory

The `.claude-plugin/` manifest files (`plugin.json`, `marketplace.json`) are **retained** for Claude Code marketplace compatibility. They are ignored by all other platforms.

---

## Retained: `src/security.ts` settings paths

The security module reads from `.claude/settings.json` files. On non-Claude platforms these files do not exist, so the module returns empty policy arrays and all commands are allowed (fail-open). No change was needed — the behavior is already gracefully platform-agnostic.
