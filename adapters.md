# Adapter System

CAP.md scripts use **generic function names** for storage operations. Each runtime provider maps these to its own implementation. This makes caps portable across different AI tool platforms.

## Adapter Functions

Scripts call these generic names:

| Generic Name | Purpose |
|-------------|---------|
| `cap_store(args)` | CRUD operations on per-cap database tables |
| `blob_put(args)` | Store large payloads (>30KB) in chunked blob storage |
| `blob_get(args)` | Retrieve chunked blob data |

## How Adapters Work

```
CAP.md script          Adapter layer           Provider API
──────────────         ─────────────           ────────────
cap_store(...)    →    mapping table     →    mcp__yesmem__cap_store(...)
blob_put(...)     →    mapping table     →    mcp__yesmem__blob_put(...)
blob_get(...)     →    mapping table     →    mcp__yesmem__blob_get(...)
```

The adapter layer is a simple name-mapping table. No logic transformation — the function signatures and parameters are identical on both sides. Only the name changes.

## Provider Mapping Example

A provider registers its adapter table at startup:

```
Generic Name  →  Provider-Specific Name
────────────     ──────────────────────
cap_store     →  mcp__yesmem__cap_store
blob_put      →  mcp__yesmem__blob_put
blob_get      →  mcp__yesmem__blob_get
```

A different provider (e.g., a Python-based memory system) would map to its own API:

```
Generic Name  →  Provider-Specific Name
────────────     ──────────────────────
cap_store     →  pymem.store
blob_put      →  pymem.blob_write
blob_get      →  pymem.blob_read
```

## Bidirectional Mapping

The adapter works in both directions:

**On save (inbound):** Provider-specific names in the script → converted to generic names before storage.
A script written as `mcp__yesmem__cap_store(...)` is normalized to `cap_store(...)` in the stored CAP.md.

**On activate (outbound):** Generic names in the stored CAP.md → converted to provider-specific names before delivery to the runtime.
`cap_store(...)` becomes `mcp__yesmem__cap_store(...)` when loaded into a YesMem session.

This means:
- The canonical CAP.md always uses generic names
- Scripts work regardless of which provider originally created them
- Migrating between providers requires zero changes to the CAP.md file

## Writing Portable Scripts

Use generic names in your scripts:

```javascript
// Portable — works with any provider
async ({ cap, table, where }) => {
    const result = await cap_store({
        capability: cap,
        action: 'query',
        table: table,
        where: where
    });
    return result;
}
```

Avoid provider-specific names:

```javascript
// NOT portable — tied to YesMem
async ({ cap, table, where }) => {
    const result = await mcp__yesmem__cap_store({  // ← don't do this
        capability: cap,
        action: 'query',
        table: table,
        where: where
    });
    return result;
}
```

The adapter layer normalizes provider-specific names on save, so existing scripts with provider names still work — but new scripts should use generic names from the start.

## cap_store API

The `cap_store` function supports these actions:

| Action | Required Params | Description |
|--------|----------------|-------------|
| `create_table` | `capability`, `table`, `columns` (JSON array of `{name, type}`) | Create a namespaced table |
| `upsert` | `capability`, `table`, `data` (JSON object) | Insert or update a row |
| `query` | `capability`, `table` | Query rows (optional: `where`, `args`, `limit`, `offset`) |
| `delete` | `capability`, `table` | Delete rows (optional: `where`, `args`) |
| `list_tables` | `capability` | List all tables for this capability |

Tables are automatically namespaced: `cap_<capability>__<table>`.

## Implementing a Provider

To support CAP.md in a new system:

1. Implement the three adapter functions (`cap_store`, `blob_put`, `blob_get`) in your runtime
2. Register a mapping table: generic name → your implementation
3. On cap load: replace generic names in the script with your function names
4. On cap save: replace your function names with generic names

The rest — frontmatter parsing, section extraction, script execution — follows the [spec](spec.md).
