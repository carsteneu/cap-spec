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

## Actions
<optional: named subsections for setup, upgrade, etc.>
```

Purpose, Script, and Database are parsed by position and heading name. Actions is optional and contains `### <Name>` subsections. Additional Markdown content outside these sections is ignored by parsers but preserved for human readers.

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
| `requires` | string[] | Auto-detected | Adapter primitives used by the script (e.g., `["store", "web"]`). |

### Runtime Detection

If `runtime` is not specified, it is inferred from the first code block in the Script section:

- ` ```javascript` or ` ```js` → `repl`
- ` ```bash` or ` ```sh` → `bash`

### Requires Detection

If `requires` is not specified, parsers scan the Script section for calls to adapter primitives:

- `store(` → adds `"store"`
- `web(` → adds `"web"`
- `file(` → adds `"file"`

### Schema Derivation

The `schema` is derived from the JavaScript function signature. `async ({ subreddit, topic, limit = 25 }) => { ... }` yields three properties — `subreddit` and `topic` required, `limit` optional with default 25. For stricter validation, the schema can be set explicitly in frontmatter.

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

> **Prerequisite:** The REPL VM must be enabled in your AI coding assistant. In Claude Code, set `CLAUDE_CODE_REPL=true` in the `env` block of your `settings.json`. See the [Claude Code REPL documentation](https://docs.anthropic.com/en/docs/claude-code) for details.
>
> **Tool availability:** Enabling REPL mode changes the tool landscape. Classic tools (`Read`, `Bash`, `Grep`, `Glob`) become REPL-internal shorthands (`cat()`, `sh()`, `rg()`, `gl()`). `Edit` and `Write` remain available as top-level tools. Cap scripts run inside the REPL VM and have access to all shorthands plus adapter primitives.

The script is a JavaScript async function expression. It receives a single destructured parameter object:

```javascript
async ({ param1, param2, optionalParam }) => {
    // ... logic ...
    return { result: "value" };
}
```

**Conventions:**
- Always async — adapter primitives (`store`, `web`, `file`) are async
- Single parameter object, destructured in signature
- Returns a plain object (JSON-serializable)
- Error case: return `{ error: "message", ...details }` instead of throwing
- Use adapter primitives for persistence, network, and file I/O (see [adapters.md](adapters.md))

### Bash Runtime (`runtime: bash`)

The script is a shell command or script:

```bash
git log --all --since=midnight --pretty=format:'%h %an %s'
```

**Conventions:**
- Receives parameters as environment variables: `$PARAM1`, `$PARAM2`
- Output to stdout is the return value
- Non-zero exit code signals error

### Headless / Automation

Caps work in non-interactive (headless) mode. In Claude Code, `claude -p "prompt"` loads the full MCP configuration including REPL, adapter primitives, and all registered caps. This enables:

- **Scheduled tasks / cron jobs** — run caps on a timer without an interactive session
- **Pipeline integration** — pipe prompts through `claude -p` in shell scripts
- **Daemon-driven execution** — a background service can invoke caps via `claude -p --project-dir <dir>`

Headless sessions have the same cap access as interactive sessions. The only difference is the absence of user interaction — the cap runs to completion and exits.

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

## 5. Actions Section

The optional `## Actions` section contains named subsections (`### <Name>`) that define provider-triggered workflows. Each subsection is a free-text block of instructions that the runtime can present to the user or agent at the appropriate time.

```markdown
## Actions

### Setup

To use this cap, you need a Telegram Bot Token:

1. Open @BotFather in Telegram
2. Send /newbot and follow the prompts
3. Copy the token and run:

store({ action: 'upsert', table: 'config', data: { key: 'telegram_bot_token', value: '<YOUR_TOKEN>' } })

After setup, this cap will stop showing these instructions.
```

### Structure

- The section heading is `## Actions` (case-sensitive)
- Each subsection is identified by a `### <Name>` heading
- Subsection content is everything between one `### ` heading and the next (or the end of `## Actions`)
- Content is preserved verbatim, including Markdown formatting and inline code

The parsed result is a map of subsection names to their content:

```
Actions["Setup"] = "To use this cap, you need a Telegram Bot Token:\n\n1. Open @BotFather..."
Actions["Upgrade"] = "..."
```

### The Setup Convention

`### Setup` is the first standardized action. It contains human-readable instructions that guide a user through first-time configuration (API keys, credentials, external accounts).

**Lifecycle:**

1. When a cap with `### Setup` is activated and no `_setup_complete` flag exists, the runtime displays the setup instructions
2. The user follows the instructions (stores credentials, creates accounts, etc.)
3. When setup is done, the runtime sets a completion flag in cap_store:
   ```javascript
   store({ action: 'upsert', table: 'config',
           data: { key: '<cap_name>_setup_complete', value: '1' } })
   ```
4. On subsequent activations, the runtime checks the flag and skips the setup block

**Rules for Setup content:**

- Instructions are free-text Markdown, not a machine-parsed schema
- Any `store()` calls in the instructions are examples for the user/agent to execute, not auto-run code
- The `_setup_complete` flag name follows the pattern `<cap_name>_setup_complete`
- A cap without `### Setup` has no first-run flow; it works immediately after activation

### Custom Actions

Beyond `### Setup`, cap authors can define arbitrary actions via additional `### <Name>` subsections. These are not interpreted by the runtime unless a provider explicitly supports them. They serve as documentation or as hooks for future provider features.

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
6. The Actions section is parsed into a `map[string]string` of `### <Name>` subsections to their content
7. Unknown sections are preserved but not interpreted
8. Markdown formatting within sections is preserved verbatim

## What Doesn't Belong in the Format

- **Tests** — belong in the script itself or in a separate test harness
- **Dependencies** — caps use the runtime's built-in utilities; if more is needed, use a classic plugin/skill
- **Permissions** — derived from `runtime` and script content at parse time
- **Author** — source attribution is handled by runtime metadata, not the cap file
