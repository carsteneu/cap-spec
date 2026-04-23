# Blob Pipe: Handling Large Payloads

AI coding assistant runtimes typically have output size limits on shell commands (commonly ~30KB). The blob pipe pattern bypasses this by streaming large payloads directly into the store primitive's blob storage, then reading them back programmatically.

## The Problem

```javascript
// This breaks on payloads >30KB — sh() truncates or errors
const data = await sh('curl -s https://api.example.com/big-endpoint');
```

## The Solution

Pipe the output directly into blob storage, then read it back in chunks:

```javascript
// 1. Pipe into blob storage (shell → blob_put)
const putResult = await sh(
    `curl -sL "https://api.example.com/big-endpoint" --max-time 15 | blob_put --cap my_cap --key "request:123"`,
    20000
);

// 2. Read back in chunks via store
let chunks = [];
for (let i = 0; i < 100; i++) {
    const r = await store({
        capability: 'my_cap',
        action: 'query',
        table: 'blobs',
        where: 'key=? AND chunk_idx=?',
        args: JSON.stringify(["request:123", i]),
        limit: 1
    });
    const rows = r?.rows || [];
    if (!rows.length) break;
    chunks.push(rows[0].data || '');
}

// 3. Reassemble
const fullPayload = chunks.join('');
const data = JSON.parse(fullPayload);
```

## How It Works

```
curl (stdout)  →  blob_put (stdin)  →  store blobs table
                                        ┌──────────────────┐
                                        │ key   │ chunk_idx │
                                        │───────│───────────│
                                        │ req:1 │ 0         │
                                        │ req:1 │ 1         │
                                        │ req:1 │ 2         │
                                        └──────────────────┘
```

`blob_put` reads stdin, splits it into chunks, and stores each chunk as a row in the cap's `blobs` table. The `key` groups chunks; `chunk_idx` preserves order.

## Blob Table Schema

The blobs table is created automatically on first use. Its schema:

```sql
CREATE TABLE cap_<name>__blobs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key TEXT,
  chunk_idx INTEGER,
  data TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Key Design

Choose keys that are idempotent — re-fetching the same resource overwrites the same key:

```
"url:https://reddit.com/r/example/post123"   -- URL as key
"file:/tmp/large-output.json"                  -- file path as key
"run:2024-01-15T10:30:00Z"                    -- timestamp as key
```

## Cleanup

Always delete old blob data before writing new data for the same key:

```javascript
// Delete old chunks first
await store({
    capability: 'my_cap',
    action: 'delete',
    table: 'blobs',
    where: 'key=?',
    args: JSON.stringify(["request:123"])
});

// Then pipe new data
await sh(`curl -sL "${url}" | blob_put --cap my_cap --key "request:123"`, 20000);
```

## When to Use Blob Pipe

| Scenario | Approach |
|----------|----------|
| Response < 30KB | Direct `sh('curl ...')` is fine |
| Response > 30KB | Use blob pipe |
| Response size unknown | Use blob pipe to be safe |
| Need to store raw response for later | Use blob pipe |
| Only need a small extract | Pipe through `jq` or `grep` first, skip blob pipe |
