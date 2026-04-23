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

The key insight is the **adapter layer**: scripts use generic function names (`cap_store`, `blob_put`, `blob_get`) that each platform maps to its own API. The CAP.md file stays the same — only the adapter mapping changes.

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

## Documentation

- [spec.md](spec.md) — Full specification
- [adapters.md](adapters.md) — Adapter system and portability
- [docs/cap-store.md](docs/cap-store.md) — cap_store API reference
- [docs/blob-pipe.md](docs/blob-pipe.md) — Handling large payloads

## Examples

- [deploy](examples/deploy/) — Simple bash capability (build & deploy)
- [cap_search](examples/cap_search/) — Generic database query tool
- [reddit_fetch](examples/reddit_fetch/) — Complex capability with blob pipe, comment parsing, and persistent storage

## Reference Implementation

[YesMem](https://github.com/carsteneu/yesmem) implements the CAP.md spec with:
- Parser and writer (`internal/capfile/`)
- Adapter layer with bidirectional name mapping
- `cap_store` backed by SQLite
- Blob pipe via chunked storage

## License

MIT
