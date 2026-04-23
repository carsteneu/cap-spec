---
name: cap_search
description: Generic search over any store table with optional auto-pagination.
version: 3
tags:
  - store
  - search
  - query
  - generic
tested: true
auto_active: true
---

## Purpose

Search any capability's stored data. Wraps `store` query with sane defaults
and optional auto-pagination. Returns `{rows, count, total, has_more, next_offset}`.

Use when you need to find rows in a cap's tables: "find all reddit posts with
score > 100", "show me the last 10 search results".

Pass `all: true` to auto-loop pagination and collect every matching row
(up to `max_rows`, default 2000).

## Script

```javascript
async ({cap, table, where, args, limit, offset, all, max_rows}) => {
  if (!cap || !table) return {error: 'cap and table required'};
  const lim = (typeof limit === 'number' && limit > 0) ? limit : 100;
  const argsJson = Array.isArray(args) ? JSON.stringify(args) : (args || '[]');
  if (!all) {
    const r = await store({
      capability: cap, action: 'query', table,
      where: where || '', args: argsJson,
      limit: lim, offset: offset || 0
    });
    return typeof r === 'string' ? JSON.parse(r) : r;
  }
  const capRows = (typeof max_rows === 'number' && max_rows > 0) ? max_rows : 2000;
  let rows = [], off = 0, total = 0;
  while (rows.length < capRows) {
    const pageLim = Math.min(100, capRows - rows.length);
    const r = await store({
      capability: cap, action: 'query', table,
      where: where || '', args: argsJson,
      limit: pageLim, offset: off
    });
    const parsed = typeof r === 'string' ? JSON.parse(r) : r;
    if (!parsed?.rows?.length) break;
    rows.push(...parsed.rows);
    total = parsed.total || total;
    if (!parsed.has_more) break;
    off = parsed.next_offset;
  }
  return {rows, count: rows.length, total, truncated: rows.length >= capRows && total > rows.length};
}
```

## Database

