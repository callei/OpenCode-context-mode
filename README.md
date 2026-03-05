# Context Mode — Multi-Platform Fork

**The other half of the context problem. Now works everywhere.**

[![License: ELv2](https://img.shields.io/badge/License-ELv2-blue.svg)](LICENSE)

> **This is a fork of [mksglu/claude-context-mode](https://github.com/mksglu/claude-context-mode)**, extended to work with multiple LLM providers and AI coding tools — not just Claude Code.

Every MCP tool call dumps raw data into your context window. A Playwright snapshot costs 56 KB. Twenty GitHub issues cost 59 KB. One access log — 45 KB. After 30 minutes, 40% of your context is gone.

Context Mode is an MCP server that sits between your AI assistant and these outputs. **315 KB becomes 5.4 KB. 98% reduction.**

---

## Supported Platforms

| Platform | Type | Status |
|----------|------|--------|
| **OpenCode** | Terminal AI coding assistant | ✅ Supported |
| **ZeroClaw** | Rust-based terminal AI coding assistant | ✅ Supported |
| **VS Code** (GitHub Copilot Chat, Cursor, Continue) | Editor extension | ✅ Supported |
| **LM Studio** | Local model runner | ✅ Supported |
| **Ollama + OpenWebUI** | Local inference + web UI | ✅ Supported |
| **Claude Code** | Anthropic's coding assistant | ✅ Supported (backward-compatible) |

---

## Install

### Quick start (any platform)

```bash
npx -y context-mode
```

Or install globally:

```bash
npm install -g context-mode
```

Then run `context-mode setup` to configure your platform interactively.

---

### OpenCode

Add to `~/.config/opencode/opencode.json`:

```json
{
  "mcp": {
    "context-mode": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "context-mode"]
    }
  }
}
```

See the full guide: [Agent-actions/OPENCODE-SETUP.md](Agent-actions/OPENCODE-SETUP.md)

---

### ZeroClaw

Add to `~/.config/zeroclaw/config.toml`:

```toml
[mcp.servers.context-mode]
command = "npx"
args = ["-y", "context-mode"]
```

See the full guide: [Agent-actions/ZEROCLAW-SETUP.md](Agent-actions/ZEROCLAW-SETUP.md)

---

### VS Code / Cursor / Continue

Create `.vscode/mcp.json` in your project:

```json
{
  "servers": {
    "context-mode": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "context-mode"],
      "env": { "PROJECT_DIR": "${workspaceFolder}" }
    }
  }
}
```

See the full guide: [Agent-actions/VSCODE-SETUP.md](Agent-actions/VSCODE-SETUP.md)

---

### LM Studio

Edit `~/.lmstudio/mcp.json`:

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

See the full guide: [Agent-actions/LM-STUDIO-SETUP.md](Agent-actions/LM-STUDIO-SETUP.md)

---

### Ollama + OpenWebUI

Edit `config/mcp.json` in your OpenWebUI data directory:

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

See the full guide: [Agent-actions/OPENWEBUI-SETUP.md](Agent-actions/OPENWEBUI-SETUP.md)

---

### Claude Code (legacy)

```bash
claude mcp add context-mode -- npx -y context-mode
```

Or as a plugin:

```bash
/plugin marketplace add mksglu/claude-context-mode
/plugin install context-mode@claude-context-mode
```

---

## What's Different in This Fork

The original `mksglu/claude-context-mode` was tightly coupled to Claude Code. This fork removes those assumptions:

- **`PROJECT_DIR`** is the single env var for project root (no Claude-specific variable)
- **`start.mjs`** no longer reads or writes Claude Code's plugin registry
- **`.mcp.json`** uses `npx -y context-mode` instead of a Claude Code plugin variable
- **`context-mode setup`** now offers OpenCode, ZeroClaw, VS Code, LM Studio, and OpenWebUI options
- **Package keywords** reflect the expanded platform support

See [Agent-actions/PLAN.md](Agent-actions/PLAN.md) for the full migration log.

---

## Tools

| Tool | What it does | Context saved |
|---|---|---|
| `batch_execute` | Run multiple commands + search multiple queries in ONE call. | 986 KB → 62 KB |
| `execute` | Run code in 11 languages. Only stdout enters context. | 56 KB → 299 B |
| `execute_file` | Process files in sandbox. Raw content never leaves. | 45 KB → 155 B |
| `index` | Chunk markdown into FTS5 with BM25 ranking. | 60 KB → 40 B |
| `search` | Query indexed content with multiple queries in one call. | On-demand retrieval |
| `fetch_and_index` | Fetch URL, detect content type (HTML/JSON/text), chunk and index. | 60 KB → 40 B |

## How the Sandbox Works

Each `execute` call spawns an isolated subprocess with its own process boundary. Scripts can't access each other's memory or state. The subprocess runs your code, captures stdout, and only that stdout enters the conversation context. The raw data — log files, API responses, snapshots — never leaves the sandbox.

Eleven language runtimes are available: JavaScript, TypeScript, Python, Shell, Ruby, Go, Rust, PHP, Perl, R, and Elixir. Bun is auto-detected for 3-5x faster JS/TS execution.

Authenticated CLIs work through credential passthrough — `gh`, `aws`, `gcloud`, `kubectl`, `docker` inherit environment variables and config paths without exposing them to the conversation.

When output exceeds 5 KB and an `intent` is provided, Context Mode switches to intent-driven filtering: it indexes the full output into the knowledge base, searches for sections matching your intent, and returns only the relevant matches with a vocabulary of searchable terms for follow-up queries.

## How the Knowledge Base Works

The `index` tool chunks markdown content by headings while keeping code blocks intact, then stores them in a **SQLite FTS5** (Full-Text Search 5) virtual table. Search uses **BM25 ranking** — a probabilistic relevance algorithm that scores documents based on term frequency, inverse document frequency, and document length normalization. **Porter stemming** is applied at index time so "running", "runs", and "ran" match the same stem.

When you call `search`, it returns relevant content snippets focused around matching query terms — not full documents, not approximations, the actual indexed content with smart extraction around what you're looking for. `fetch_and_index` extends this to URLs: fetch, convert HTML to markdown, chunk, index. The raw page never enters context.

## Fuzzy Search

Search uses a three-layer fallback to handle typos, partial terms, and substring matches:

- **Layer 1 — Porter stemming**: Standard FTS5 MATCH with porter tokenizer. "caching" matches "cached", "caches", "cach".
- **Layer 2 — Trigram substring**: FTS5 trigram tokenizer matches partial strings. "useEff" finds "useEffect", "authenticat" finds "authentication".
- **Layer 3 — Fuzzy correction**: Levenshtein distance corrects typos before re-searching. "kuberntes" → "kubernetes", "autentication" → "authentication".

The `searchWithFallback` method cascades through all three layers and annotates results with `matchLayer` so you know which layer resolved the query.

## Smart Snippets

Search results use intelligent extraction instead of truncation. Instead of returning the first N characters (which might miss the important part), Context Mode finds where your query terms appear in the content and returns windows around those matches. If your query is "authentication JWT token", you get the paragraphs where those terms actually appear — not an arbitrary prefix.

## Progressive Search Throttling

The `search` tool includes progressive throttling to prevent context flooding from excessive individual calls:

- **Calls 1-3:** Normal results (2 per query)
- **Calls 4-8:** Reduced results (1 per query) + warning
- **Calls 9+:** Blocked — redirects to `batch_execute`

This encourages batching queries via `search(queries: ["q1", "q2", "q3"])` or `batch_execute` instead of making dozens of individual calls.

## Session Stats

The `stats` tool tracks context consumption in real-time.

| Metric | Value |
|---|---|
| Session | 1.4 min |
| Tool calls | 1 |
| Total data processed | **9.6MB** |
| Kept in sandbox | **9.6MB** |
| Entered context | 0.3KB |
| Tokens consumed | ~82 |
| **Context savings** | **24,576.0x (99% reduction)** |

## The Numbers

Measured across real-world scenarios:

**Playwright snapshot** — 56.2 KB raw → 299 B context (99% saved)
**GitHub Issues (20)** — 58.9 KB raw → 1.1 KB context (98% saved)
**Access log (500 requests)** — 45.1 KB raw → 155 B context (100% saved)
**Context7 React docs** — 5.9 KB raw → 261 B context (96% saved)
**Analytics CSV (500 rows)** — 85.5 KB raw → 222 B context (100% saved)
**Git log (153 commits)** — 11.6 KB raw → 107 B context (99% saved)
**Test output (30 suites)** — 6.0 KB raw → 337 B context (95% saved)
**Repo research (subagent)** — 986 KB raw → 62 KB context (94% saved, 5 calls vs 37)

Over a full session: 315 KB of raw output becomes 5.4 KB.

[Full benchmark data with 21 scenarios →](BENCHMARK.md)

## Security

Context Mode enforces permission rules using the same format as Claude Code's `settings.json` — but the security layer works on any platform. If you block `sudo`, it's also blocked inside `execute`, `execute_file`, and `batch_execute`.

**Zero setup required.** If you haven't configured any permissions, all commands are permitted. This only activates when you add rules.

### Rule format

Create `.claude/settings.json` in your project root (or `~/.claude/settings.json` globally):

```json
{
  "permissions": {
    "deny": [
      "Bash(sudo *)",
      "Bash(rm -rf /*)",
      "Read(.env)",
      "Read(**/.env*)"
    ],
    "allow": [
      "Bash(git:*)",
      "Bash(npm:*)"
    ]
  }
}
```

The pattern is `Tool(what to match)` where `*` means "anything". Rules are checked in order: project-local → project-shared → global. `deny` always wins over `allow`.

## Environment Variables

| Variable | Platform | Description |
|----------|----------|-------------|
| `PROJECT_DIR` | All platforms | Project root for security policy resolution |

## Requirements

- **Node.js 18+**
- Optional: **Bun** (auto-detected, 3-5× faster JS/TS)

## Contributing

```bash
git clone https://github.com/callei/OpenCode-context-mode.git
cd OpenCode-context-mode && npm install
npm test              # run tests
npm run test:all      # full suite
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full local development workflow.

## License

[Elastic License 2.0 (ELv2)](LICENSE) — free to use, modify, and share. You may not rebrand and redistribute this software as a competing plugin, product, or managed service.

