# Adapter System

Adapters are the portability layer of the CAP.md format. A capability script uses **3 generic primitive functions** instead of provider-specific tool calls. Each provider maps these primitives to its own implementation at activation time.

## The 3 Primitives

Every adapter is action-based: one function, multiple operations via the `action` parameter.

### `store(...)` — Persistence

All data operations: structured tables and large blob storage.

| Action | Description |
|--------|-------------|
| `create_table` | Create a table with given columns |
| `upsert` | Insert or update a row |
| `query` | Read rows with optional WHERE clause |
| `delete` | Remove rows |
| `list_tables` | List all tables for this capability |
| `blob_put` | Store large data (>30KB) in chunks |
| `blob_get` | Retrieve chunked blob data |

```javascript
// Structured data
await store({ capability: 'my_cap', action: 'upsert', table: 'hits',
  data: { post_id: '123', title: 'Hello', score: 42 } });

// Query with filter
const rows = await store({ capability: 'my_cap', action: 'query', table: 'hits',
  where: 'score > ?', args: '[10]', limit: 50 });

// Large payload
await store({ capability: 'my_cap', action: 'blob_put', table: 'blobs',
  data: JSON.stringify(largeObject) });
```

### `web(...)` — Network I/O

HTTP requests and web search.

| Action | Description |
|--------|-------------|
| `fetch` | Fetch a URL, return raw content |
| `search` | Web search, return results |

```javascript
// Fetch a page
const page = await web({ action: 'fetch', url: 'https://example.com/api/data',
  prompt: 'Extract the version number' });

// Search the web
const results = await web({ action: 'search', query: 'CAP.md format specification' });
```

### `file(...)` — Filesystem I/O

Local file operations.

| Action | Description |
|--------|-------------|
| `read` | Read file content |
| `write` | Write content to a file |
| `glob` | Find files matching a pattern |

```javascript
// Read a file
const content = await file({ action: 'read', path: '/home/user/data.json' });

// Write a file
await file({ action: 'write', path: '/tmp/report.md', content: reportText });

// Find files
const matches = await file({ action: 'glob', pattern: 'src/**/*.ts' });
```

## What is NOT an Adapter

Runtime builtins are always available regardless of provider. They are not adapters:

| Builtin | Purpose |
|---------|---------|
| `sh(cmd)` | Shell execution |
| `haiku(prompt, schema?)` | LLM inference |
| `log(...)` | Console output |
| `JSON.parse/stringify` | Serialization |

These are part of the execution environment, not the portability layer.

## Provider Mapping

Each provider registers a mapping from generic primitives to its own implementation:

### YesMem (Reference Implementation)

| Primitive | Provider Implementation |
|-----------|----------------------|
| `store(...)` | `mcp__yesmem__cap_store(...)` |
| `web({ action: 'fetch' })` | `sh('curl -sL --max-time 20 ' + shQuote(url))` |
| `web({ action: 'search' })` | `WebSearch(...)` |
| `file({ action: 'read' })` | `cat(path)` |
| `file({ action: 'write' })` | `put(path, content)` |
| `file({ action: 'glob' })` | `gl(pattern, path)` |

### Other Providers

A minimal provider needs only to implement the 3 functions:

```javascript
// Provider registration (pseudocode)
registerAdapter('store', async (params) => {
  // route params.action to your storage backend
});
registerAdapter('web', async (params) => {
  // route params.action to your HTTP client
});
registerAdapter('file', async (params) => {
  // route params.action to your filesystem API
});
```

## Roundtrip Flow

1. **Save** (`ProviderToGeneric`): when saving a cap, provider-specific calls in the script are replaced with generic names. `mcp__yesmem__cap_store(` → `store(`
2. **Store**: the CAP.md file on disk always contains generic names only.
3. **Activate** (`GenericToProvider`): when loading a cap for execution, generic names are replaced with the current provider's implementation. `store(` → `mcp__yesmem__cap_store(`

This ensures caps are portable: a cap written in YesMem can be activated in any provider that implements the 3 primitives.

## Design Principles

- **Action-based dispatch**: one function per I/O category, actions via parameter. Not one function per operation.
- **Minimal surface**: 3 primitives cover all I/O categories (persistence, network, filesystem). Everything else is a runtime builtin.
- **No adapter for inference**: `haiku()` stays a runtime builtin. LLM calls are not provider-portable in the same way — model names, parameters, and capabilities differ too much.
- **No adapter for shell**: `sh()` is universal. Every execution environment has shell access.
