# CAP.md Format Specification

Version: 1.0-draft

## Overview

A CAP.md file is a Markdown document with YAML frontmatter that defines a single reusable tool (capability). It is designed to be read by both humans and AI assistants, enabling any LLM-based coding tool to parse the definition and execute or translate it.

## File Structure

```
---
<frontmatter: YAML metadata>
---

## Purpose
<what the tool does, when to use it>

## Script
<code block with the executable logic>

## Database
<optional: SQL schema for persistent storage>
```

All four sections are parsed by position and heading name. Additional Markdown content outside these sections is ignored by parsers but preserved for human readers.

## 1. Frontmatter

YAML block between `---` delimiters. Contains tool metadata.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Unique identifier. Lowercase, underscores. Must match the directory name. |
| `description` | string | One-line description of what the tool does. Shown in tool listings. |

### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | integer | `1` | Auto-incremented on save. Tracks revisions. |
| `tags` | string[] | `[]` | Categorization tags for filtering and discovery. |
| `runtime` | `"repl"` \| `"bash"` | Detected from script language | Execution environment. |
| `scope` | `"user"` \| `"project"` | `"user"` | Visibility scope. User-scoped caps are available everywhere; project-scoped only in that project. |
| `tested` | boolean | `false` | Whether the handler has been verified working. |
| `auto_active` | boolean | `false` | When true, tool is activated automatically at session start. |
| `requires` | string[] | Auto-detected | Generic adapter functions used by the script (e.g., `["cap_store", "blob_put"]`). |

### Runtime Detection

If `runtime` is not specified, it is inferred from the first code block in the Script section:

- ` ```javascript` or ` ```js` → `repl`
- ` ```bash` or ` ```sh` → `bash`

### Requires Detection

If `requires` is not specified, parsers scan the Script section for calls to adapter functions:

- `cap_store(` → adds `"cap_store"`
- `blob_put(` → adds `"blob_put"`
- `blob_get(` → adds `"blob_get"`

## 2. Purpose Section

Starts with `## Purpose`. Free-form Markdown describing:

- What the tool does
- When to use it
- Expected inputs and outputs
- Edge cases or limitations

This section is the primary documentation for both human readers and AI assistants deciding whether to use the tool.

## 3. Script Section

Starts with `## Script`. Contains exactly one fenced code block with the executable logic.

### REPL Runtime (`runtime: repl`)

The script is a JavaScript async function expression. It receives a single destructured parameter object:

```javascript
async ({ param1, param2, optionalParam }) => {
    // ... logic ...
    return { result: "value" };
}
```

**Conventions:**
- Always async — adapter functions are async
- Single parameter object, destructured in signature
- Returns a plain object (JSON-serializable)
- Error case: return `{ error: "message", ...details }` instead of throwing
- Use adapter functions for storage (see [adapters.md](adapters.md))

### Bash Runtime (`runtime: bash`)

The script is a shell command or script:

```bash
git log --all --since=midnight --pretty=format:'%h %an %s'
```

**Conventions:**
- Receives parameters as environment variables: `$PARAM1`, `$PARAM2`
- Output to stdout is the return value
- Non-zero exit code signals error

## 4. Database Section

Starts with `## Database`. Optional. Contains SQL `CREATE TABLE` statements defining the cap's persistent storage schema.

### Table Naming

Tables follow the convention: `cap_<capability_name>__<table_name>`

```sql
CREATE TABLE cap_reddit_fetch__posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  permalink TEXT,
  title TEXT,
  score INTEGER,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

The double underscore `__` separates the capability namespace from the table name.

### Standard Columns

Every table gets these columns automatically (do not omit them):

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Row identifier |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | Row creation time |
| `updated_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | Last modification time |

### Blob Storage

For large payloads, a special `blobs` table is used (see [docs/blob-pipe.md](docs/blob-pipe.md)):

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

### Empty Database Section

If a cap uses no persistent storage, include the heading with empty content:

```markdown
## Database

```

This signals explicitly that the cap is stateless.

## File System Layout

```
~/.claude/caps/
└── <cap_name>/
    └── CAP.md
```

Each capability lives in its own directory. The directory name must match the `name` field in frontmatter.

## Parsing Rules

1. Frontmatter is extracted between the first pair of `---` lines
2. Sections are identified by `## <Name>` headings (case-sensitive)
3. Code blocks are extracted from within their section by matching ` ``` ` fences
4. The first code block in the Script section is the handler
5. The first code block in the Database section (if any) contains the schema
6. Unknown sections are preserved but not interpreted
7. Markdown formatting within sections is preserved verbatim
