# ZeroClaw Setup Guide

## What is ZeroClaw?

[ZeroClaw](https://github.com/zeroclaw/zeroclaw) is a Rust-based terminal AI coding assistant and the OpenClaw alternative — a high-performance, low-latency AI coding tool that supports the Model Context Protocol (MCP) natively. Because it is written in Rust, ZeroClaw starts instantly and uses minimal memory.

---

## Prerequisites

- ZeroClaw installed (see [ZeroClaw releases](https://github.com/zeroclaw/zeroclaw/releases))
- Node.js 18+ installed on your system

---

## Installation

### Option 1: npx (no install required)

Add to your ZeroClaw configuration at `~/.config/zeroclaw/config.toml`:

```toml
[mcp.servers.context-mode]
command = "npx"
args = ["-y", "context-mode"]
```

### Option 2: Global npm install

```bash
npm install -g context-mode
```

Then in `~/.config/zeroclaw/config.toml`:

```toml
[mcp.servers.context-mode]
command = "context-mode"
```

### Option 3: Local clone

```bash
git clone https://github.com/callei/OpenCode-context-mode
cd OpenCode-context-mode
npm install
npm run build
```

Then in `~/.config/zeroclaw/config.toml`:

```toml
[mcp.servers.context-mode]
command = "node"
args = ["/path/to/OpenCode-context-mode/build/server.js"]
```

---

## Environment Variables

You can pass environment variables to the MCP server in your ZeroClaw config:

```toml
[mcp.servers.context-mode]
command = "npx"
args = ["-y", "context-mode"]

[mcp.servers.context-mode.env]
PROJECT_DIR = "/path/to/your/project"
```

| Variable | Default | Description |
|----------|---------|-------------|
| `PROJECT_DIR` | current working directory | Set to your project root for security policy resolution |

ZeroClaw sets the working directory to your project root when launching MCP servers, so you typically do not need to set `PROJECT_DIR` manually.

---

## Available Tools

Once configured, ZeroClaw can call these MCP tools:

| Tool | Description |
|------|-------------|
| `batch_execute` | Run multiple shell commands + search all output in one call |
| `execute` | Run code in 11 languages in a sandboxed subprocess |
| `execute_file` | Process files in sandbox without loading contents into context |
| `index` | Chunk and index markdown/docs into FTS5 knowledge base |
| `search` | Query indexed content with BM25 ranking |
| `fetch_and_index` | Fetch a URL, convert to markdown, index into knowledge base |
| `stats` | Show context savings statistics for the current session |

---

## Verification

Start ZeroClaw in your project directory. The MCP server should appear in the connected servers list. You can verify it is running by asking:

```
Use the context-mode stats tool to show session statistics.
```

---

## Performance Tip

Install [Bun](https://bun.sh) for 3–5× faster JavaScript/TypeScript execution inside the sandbox:

```bash
curl -fsSL https://bun.sh/install | bash
```

---

## Notes

- ZeroClaw communicates with MCP servers over **stdio** — no network ports are required
- The `execute` sandbox inherits your shell environment so authenticated CLIs (`gh`, `aws`, `kubectl`) work without extra setup
- Security policies (if configured) are read from `.claude/settings.json` or `~/.claude/settings.json`; on fresh setups all commands are permitted (fail-open)
