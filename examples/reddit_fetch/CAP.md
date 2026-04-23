---
name: reddit_fetch
description: Fetch a Reddit post URL and return structured data with comments, links, and persistent storage.
version: 5
tags:
  - web
  - reddit
  - fetch
  - structured-storage
runtime: repl
tested: true
requires:
  - cap_store
---

## Purpose

Fetch a Reddit post and extract structured data: post metadata, nested comments
(with depth and threading), and unique external links categorized by domain
(github/reddit/external).

Data is also persisted into cap_store tables for later querying:
- `posts` — one row per fetched post
- `comments` — one row per comment, preserving depth and parent relationships
- `links` — one row per unique external URL, with categorization

Uses the blob pipe pattern to handle Reddit's large JSON responses (often >100KB)
without hitting shell output limits.

Accepts URLs with or without `reddit:` prefix. Pass `max_comments` to limit
comment extraction.

## Script

```javascript
async ({url, max_comments}) => {
  if (!url || typeof url !== 'string') return {error: 'url required (string)'};
  url = url.replace(/^reddit:/i, '').trim().replace(/\/$/, '');
  if (!/^https?:\/\/(www\.|old\.)?reddit\.com\//i.test(url))
    return {error: 'not a reddit URL', given: url};

  const fetchUrl = url + '.json?limit=500&raw_json=1';
  const key = 'url:' + url;

  // Fetch via blob pipe — bypasses 30KB shell limit
  const putRes = await sh(
    `curl -sL -A "YesMem/1.0" ${JSON.stringify(fetchUrl)} --max-time 15`
    + ` | blob_put --cap reddit_fetch --key ${JSON.stringify(key)}`,
    20000
  );
  if (!putRes || !putRes.includes('"status":"ok"'))
    return {error: 'blob_put failed', detail: String(putRes).slice(0, 400)};

  // Read back chunks
  let rows = [];
  for (let i = 0; i < 50; i++) {
    const r = await cap_store({
      capability: 'reddit_fetch', action: 'query', table: 'blobs',
      where: 'key=? AND chunk_idx=?', args: JSON.stringify([key, i]), limit: 1
    });
    const parsed = typeof r === 'string' ? JSON.parse(r) : r;
    const arr = Array.isArray(parsed) ? parsed : (parsed.rows || []);
    if (!arr.length) break;
    rows.push(arr[0]);
  }
  if (!rows.length) return {error: 'blob empty after put', key};

  const raw = rows.map(r => r.data || '').join('');
  let data;
  try { data = JSON.parse(raw); }
  catch (e) { return {error: 'invalid json', preview: raw.slice(0, 300)}; }
  if (!Array.isArray(data) || data.length < 2) return {error: 'unexpected shape'};

  const postData = data[0]?.data?.children?.[0]?.data;
  if (!postData) return {error: 'no post in response'};

  const permalink = `https://reddit.com${postData.permalink}`;
  const fetchedAt = Math.floor(Date.now() / 1000);

  // Ensure tables exist
  const cols = {
    posts: [{name:'permalink',type:'TEXT'},{name:'subreddit',type:'TEXT'},
            {name:'author',type:'TEXT'},{name:'title',type:'TEXT'},
            {name:'body',type:'TEXT'},{name:'score',type:'INTEGER'},
            {name:'num_comments',type:'INTEGER'},{name:'created_utc',type:'INTEGER'},
            {name:'external_url',type:'TEXT'},{name:'fetched_at',type:'INTEGER'}],
    comments: [{name:'post_permalink',type:'TEXT'},{name:'comment_id',type:'TEXT'},
               {name:'depth',type:'INTEGER'},{name:'author',type:'TEXT'},
               {name:'score',type:'INTEGER'},{name:'body',type:'TEXT'},
               {name:'created_utc',type:'INTEGER'},{name:'parent_id',type:'TEXT'},
               {name:'fetched_at',type:'INTEGER'}],
    links: [{name:'post_permalink',type:'TEXT'},{name:'target_url',type:'TEXT'},
            {name:'kind',type:'TEXT'},{name:'source_kind',type:'TEXT'},
            {name:'source_author',type:'TEXT'},{name:'source_comment_id',type:'TEXT'},
            {name:'fetched_at',type:'INTEGER'}]
  };
  for (const [t, c] of Object.entries(cols)) {
    await cap_store({capability:'reddit_fetch', action:'create_table', table:t,
                     columns:JSON.stringify(c)});
  }

  // Clear old data for this post
  for (const t of ['posts','comments','links']) {
    const w = t === 'posts' ? 'permalink=?' : 'post_permalink=?';
    await cap_store({capability:'reddit_fetch', action:'delete', table:t,
                     where:w, args:JSON.stringify([permalink])});
  }

  // Store post
  await cap_store({capability:'reddit_fetch', action:'upsert', table:'posts',
    data:JSON.stringify({
      permalink, subreddit:postData.subreddit||'', author:postData.author||'[deleted]',
      title:postData.title||'', body:postData.selftext||'', score:postData.score||0,
      num_comments:postData.num_comments||0, created_utc:Math.floor(postData.created_utc||0),
      external_url:(postData.url&&!postData.is_self)?postData.url:'', fetched_at:fetchedAt
    })});

  // Link extraction
  const categorize = (u) => {
    const m = u.match(/^https?:\/\/([^/?#:]+)/i);
    if (!m) return 'external';
    const host = m[1].toLowerCase();
    if (host === 'github.com' || host.endsWith('.github.com')) return 'github';
    if (host === 'reddit.com' || host.endsWith('.reddit.com') || host === 'redd.it') return 'reddit';
    return 'external';
  };
  const linkSet = new Set();
  const linkRows = [];
  const urlRe = /https?:\/\/[^\s\)\]\>"'<]+/g;
  const collect = (text, sourceKind, author, cid) => {
    if (!text) return;
    const m = text.match(urlRe);
    if (!m) return;
    for (const u of m) {
      if (linkSet.has(u)) continue;
      linkSet.add(u);
      linkRows.push({post_permalink:permalink, target_url:u, kind:categorize(u),
                      source_kind:sourceKind, source_author:author||'',
                      source_comment_id:cid||'', fetched_at:fetchedAt});
    }
  };
  collect(postData.selftext, 'post_body', postData.author, '');

  // Comment extraction with depth tracking
  const comments = [];
  const commentRows = [];
  const cap = typeof max_comments === 'number' && max_comments > 0 ? max_comments : 0;
  const walk = (children, depth) => {
    for (const c of children) {
      if (cap && comments.length >= cap) return;
      if (!c?.data || c.kind === 'more') continue;
      const d = c.data;
      if (!d.body) continue;
      comments.push({author:d.author, score:d.score, depth, body:d.body});
      commentRows.push({post_permalink:permalink, comment_id:d.id||'', depth,
                         author:d.author||'[deleted]', score:d.score||0, body:d.body,
                         created_utc:Math.floor(d.created_utc||0), parent_id:d.parent_id||'',
                         fetched_at:fetchedAt});
      collect(d.body, 'comment', d.author, d.id);
      const rc = d.replies?.data?.children;
      if (Array.isArray(rc)) walk(rc, depth + 1);
    }
  };
  walk(data[1]?.data?.children ?? [], 0);

  // Persist comments and links
  for (const row of commentRows) {
    await cap_store({capability:'reddit_fetch', action:'upsert', table:'comments',
                     data:JSON.stringify(row)});
  }
  for (const row of linkRows) {
    await cap_store({capability:'reddit_fetch', action:'upsert', table:'links',
                     data:JSON.stringify(row)});
  }

  return {
    post: {title:postData.title, author:postData.author, score:postData.score,
           subreddit:postData.subreddit, permalink, body:postData.selftext||''},
    comments,
    links: Array.from(linkSet),
    stats: {comment_count:comments.length, link_count:linkSet.size,
            reported_comments:postData.num_comments},
    stored: {posts:1, comments:commentRows.length, links:linkRows.length}
  };
}
```

## Database

```sql
CREATE TABLE cap_reddit_fetch__blobs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key TEXT,
  chunk_idx INTEGER,
  data TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cap_reddit_fetch__posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  permalink TEXT,
  subreddit TEXT,
  author TEXT,
  title TEXT,
  body TEXT,
  score INTEGER,
  num_comments INTEGER,
  created_utc INTEGER,
  external_url TEXT,
  fetched_at INTEGER,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cap_reddit_fetch__comments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  post_permalink TEXT,
  comment_id TEXT,
  depth INTEGER,
  author TEXT,
  score INTEGER,
  body TEXT,
  created_utc INTEGER,
  parent_id TEXT,
  fetched_at INTEGER,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE cap_reddit_fetch__links (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  post_permalink TEXT,
  target_url TEXT,
  kind TEXT,
  source_kind TEXT,
  source_author TEXT,
  source_comment_id TEXT,
  fetched_at INTEGER,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```
