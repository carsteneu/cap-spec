# CAP.md Specification

An open format for defining portable, reusable AI tool capabilities.

## What is CAP.md?

A CAP.md file defines a single tool that an AI coding assistant can use. It contains:

- **Metadata** (YAML frontmatter) — name, description, tags, runtime
- **Purpose** — what the tool does and when to use it
- **Script** — executable code (JavaScript or Bash)
- **Database** — optional SQL schema for persistent storage

## Why?

AI coding tools (Claude Code, Cursor, Windsurf, Codex) can use custom tools, but each platform has its own format. CAP.md provides a shared spec so tools can be written once and used anywhere.

The key insight is the **adapter layer**: scripts use 3 generic primitives (`store`, `web`, `file`) that each platform maps to its own API. The CAP.md file stays the same — only the adapter mapping changes.

## Quick Start

A minimal capability (`hello/CAP.md`):

````markdown
---
name: hello
description: Say hello to a user by name.
runtime: repl
---

## Purpose

Greets the user. Pass `name` to personalize the greeting.

## Script

```javascript
async ({ name }) => {
    return { message: `Hello, ${name || 'world'}!` };
}
```

## Database

````

## Prerequisites

For `runtime: repl` capabilities, the AI coding assistant must support a JavaScript REPL VM.

**Claude Code:** Set `CLAUDE_CODE_REPL=true` in the `env` block of your `settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_REPL": "true"
  }
}
```

See the [Claude Code REPL documentation](https://docs.anthropic.com/en/docs/claude-code) for setup details.

**Tool availability under REPL mode:** Classic tools (`Read`, `Bash`, `Grep`, `Glob`) become REPL-internal shorthands (`cat()`, `sh()`, `rg()`, `gl()`). `Edit` and `Write` remain top-level. Cap scripts run inside the REPL VM with access to all shorthands plus adapter primitives.

For `runtime: bash` capabilities, no special configuration is needed.

### Headless / Automation

Caps work in non-interactive mode. `claude -p "prompt"` loads the full MCP configuration — REPL, adapters, and all registered caps are available. This enables scheduled tasks, cron jobs, and pipeline integration without an interactive session.

## Documentation

- [spec.md](spec.md) — Full specification
- [adapters.md](adapters.md) — Adapter system and portability
- [docs/store.md](docs/store.md) — store primitive API reference
- [docs/blob-pipe.md](docs/blob-pipe.md) — Handling large payloads

## Examples

- [deploy](examples/deploy/) — Simple bash capability (build & deploy)
- [cap_search](examples/cap_search/) — Generic database query tool
- [reddit_fetch](examples/reddit_fetch/) — Complex capability with blob pipe, comment parsing, and persistent storage

## Reference Implementation

[YesMem](https://github.com/carsteneu/yesmem) implements the CAP.md spec with:
- Parser and writer (`internal/capfile/`)
- Adapter layer with bidirectional name mapping for 3 primitives (`store`, `web`, `file`)
- `store` backed by SQLite
- Blob pipe via chunked storage

## License

MIT
