# store: Per-Capability Database

Capabilities can store structured data persistently using the `store` adapter primitive. Each capability gets its own namespace — tables from one cap never collide with another.

## Table Naming Convention

```
cap_<capability_name>__<table_name>
```

The double underscore `__` separates namespace from table name. Example: capability `reddit_fetch` with table `posts` → `cap_reddit_fetch__posts`.

## Operations

### create_table

Creates a table if it doesn't exist. Columns are specified as a JSON array of `{name, type}` objects.

```javascript
await store({
    capability: 'my_cap',
    action: 'create_table',
    table: 'items',
    columns: JSON.stringify([
        { name: 'title', type: 'TEXT' },
        { name: 'score', type: 'INTEGER' },
        { name: 'url', type: 'TEXT' }
    ])
});
```

Standard columns (`id`, `created_at`, `updated_at`) are added automatically.

SQLite types: `TEXT`, `INTEGER`, `REAL`, `BLOB`.

### upsert

Insert or update a row. Pass data as a JSON object. Include `id` to update an existing row.

```javascript
await store({
    capability: 'my_cap',
    action: 'upsert',
    table: 'items',
    data: JSON.stringify({ title: 'Example', score: 42, url: 'https://example.com' })
});
```

### query

Read rows with optional filtering and pagination.

```javascript
const result = await store({
    capability: 'my_cap',
    action: 'query',
    table: 'items',
    where: 'score > ?',
    args: JSON.stringify([10]),
    limit: 50,
    offset: 0
});
// result: { rows: [...], count, total, has_more, next_offset }
```

The `where` clause uses SQLite syntax with `?` placeholders. Bind values are passed as a JSON array via `args`.

### delete

Remove rows matching a condition.

```javascript
await store({
    capability: 'my_cap',
    action: 'delete',
    table: 'items',
    where: 'score < ?',
    args: JSON.stringify([0])
});
```

Omitting `where` deletes all rows in the table.

### list_tables

List all tables belonging to a capability.

```javascript
const tables = await store({
    capability: 'my_cap',
    action: 'list_tables'
});
// tables: ["items", "blobs", ...]
```

## Design Guidelines

- **One table per concern.** Don't overload a single table with unrelated data.
- **Flat schemas preferred.** Store denormalized data rather than building complex joins.
- **Timestamps for freshness.** Use `fetched_at` or similar columns so stale data can be identified.
- **Idempotent writes.** Delete-then-upsert is the standard pattern for refreshing data.
- **Clean up after yourself.** If a cap fetches and stores data, old data for the same key should be replaced, not accumulated.
