# CAP.md Format Specification

Version: 1.1-draft

## Overview

A CAP.md file is a Markdown document with YAML frontmatter that defines a reusable tool (capability). A capability exposes one or more named scripts that share state (database tables) and lifecycle (setup, upgrade actions). It is designed to be read by both humans and AI assistants, enabling any LLM-based coding tool to parse the definition and execute or translate it.

## File Structure

```
---
<frontmatter: YAML metadata>
---

## Purpose
<what the capability does, when to use it>

## Scripts
### <script_name_a>
<inline metadata>
<code block>

### <script_name_b>
<inline metadata>
<code block>

## Database
<optional: SQL schema for persistent storage shared by all scripts>

## Actions
<optional: named subsections for setup, upgrade, etc.>
```

Purpose, Scripts, Database, and Actions are parsed by position and heading name. Scripts contains one or more `### <name>` subsections, each a callable function. Database and Actions are optional. Additional Markdown content outside these sections is ignored by parsers but preserved for human readers.

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
| `scope` | `"user"` \| `"project"` | `"user"` | Visibility scope. User-scoped caps are available everywhere; project-scoped only in that project. |
| `tested` | boolean | `false` | Whether the cap has been verified working. |
| `auto_active` | boolean | `false` | When true, the cap's tool-kind scripts are activated automatically at session start. |
| `requires` | string[] | Auto-detected | Adapter primitives used by any script in the cap (e.g., `["store", "web"]`). |

> Note: `runtime` and `schema` are per-script, not cap-level. See [Script Metadata](#script-metadata) below.

### Runtime Detection (per script)

Each script's runtime is inferred from its code fence language tag:

- ` ```javascript` or ` ```js` → `repl`
- ` ```bash` or ` ```sh` → `bash`

Inline `runtime:` metadata between the `### <name>` heading and the code fence overrides detection (rarely needed).

### Requires Detection

If `requires` is not specified at the cap level, parsers scan all scripts for adapter primitive calls:

- `store(` → adds `"store"`
- `web(` → adds `"web"`
- `file(` → adds `"file"`

### Schema Derivation (per tool-script)

For tool-kind scripts in REPL runtime, the JSON schema is derived from the JavaScript function signature. `async ({ subreddit, topic, limit = 25 }) => { ... }` yields three properties — `subreddit` and `topic` required, `limit` optional with default 25. For stricter validation, set `schema:` as inline metadata on the script.

## 2. Purpose Section

Starts with `## Purpose`. Free-form Markdown describing:

- What the tool does
- When to use it
- Expected inputs and outputs
- Edge cases or limitations

This section is the primary documentation for both human readers and AI assistants deciding whether to use the tool.

## 3. Scripts Section

Starts with `## Scripts`. Contains one or more named `### <name>` subsections, each defining a callable function. All scripts in a cap share the same `## Database` schema and `## Actions` lifecycle.

### Single-Script Cap

Most caps expose exactly one function. By convention the script name matches the cap name:

````markdown
## Scripts

### reddit_fetch
kind: tool

```javascript
async ({ url }) => { ... }
```
````

### Bundle Cap (Multi-Script)

A cap can bundle multiple related functions sharing data tables and configuration. Each function is a `### <name>` subsection. Different scripts may use different runtimes within the same cap.

````markdown
## Scripts

### telegram_send
kind: tool

```bash
curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" ...
```

### telegram_poll
kind: handler

```bash
... poll updates from getUpdates ...
```

### telegram_reply
kind: handler

```bash
... reply to unprocessed messages ...
```
````

### Script Metadata

Between the `### <name>` heading and the code fence, optional `key: value` lines define per-script behavior:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `kind` | `"tool"` \| `"handler"` | `"tool"` | How the script is invoked. **Tools** are registered as MCP tools and called by AI assistants. **Handlers** run via the scheduler/cron and are not exposed to AI assistants. |
| `runtime` | `"repl"` \| `"bash"` | derived from code fence | Execution environment. Override only when the code fence cannot disambiguate (rare). |
| `schema` | JSON (single line) | derived from JS signature (REPL only) | JSON Schema for tool input parameters. Tool-kind scripts only. |

**Metadata block rules:**

- The block starts on the line immediately after `### <name>`.
- The block consists of contiguous `key: value` lines. The first blank line, prose line, or code fence ends the block.
- `schema` must be a single-line JSON object (no embedded newlines). For schemas too complex to fit on one line, derive from the JavaScript signature (REPL) or split the script into multiple smaller scripts.
- Unknown keys in the metadata block cause a parse error. Implementations must not silently ignore unrecognized fields.
- For `kind: tool` with `runtime: bash`, the `schema` field is required — bash has no signature to derive from.
- For `kind: handler`, the `schema` field is invalid (handlers are not invoked with structured arguments). Setting it is a parse error.

### Script Naming

Script names follow cap name rules: lowercase, digits, underscores; must start with a letter.

For **tool**-kind scripts, the name becomes the MCP tool identifier (e.g., `reddit_fetch`, `telegram_send`). Within one cap, all script names must be unique.

For **handler**-kind scripts, the scheduler resolves a handler reference by `(cap_name, script_name)`. A scheduled job stores both: the cap directory and the script subsection name. At fire time, the runtime loads the cap, finds the matching `### <script_name>` subsection, and executes its body. Naming a non-existent script in a scheduled job is a runtime error, not a parse error (the cap may parse successfully even if a downstream scheduler entry is stale).

### REPL Runtime (`runtime: repl`)

> **Prerequisite:** The REPL VM must be enabled in your AI coding assistant. In Claude Code, set `CLAUDE_CODE_REPL=true` in the `env` block of your `settings.json`. See the [Claude Code REPL documentation](https://docs.anthropic.com/en/docs/claude-code) for details.
>
> **Tool availability:** Enabling REPL mode changes the tool landscape. Classic tools (`Read`, `Bash`, `Grep`, `Glob`) become REPL-internal shorthands (`cat()`, `sh()`, `rg()`, `gl()`). `Edit` and `Write` remain available as top-level tools. Cap scripts run inside the REPL VM and have access to all shorthands plus adapter primitives.

A REPL script is a JavaScript async function expression. It receives a single destructured parameter object:

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

A bash script is a shell command or script:

```bash
git log --all --since=midnight --pretty=format:'%h %an %s'
```

**Conventions:**
- Receives parameters as environment variables: `$PARAM1`, `$PARAM2`
- Output to stdout is the return value
- Non-zero exit code signals error
- Handler-kind bash scripts have access to `store` (CLI binding) for adapter primitives but no MCP runtime

### Headless / Automation

Caps work in non-interactive (headless) mode. In Claude Code, `claude -p "prompt"` loads the full MCP configuration including REPL, adapter primitives, and all registered caps. This enables:

- **Scheduled tasks / cron jobs** — handler-kind scripts run on a timer without an interactive session
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

### Stateless Caps

If a cap uses no persistent storage, omit the `## Database` section entirely. The absence of the heading signals a stateless cap. Renderers must not emit an empty `## Database` heading.

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
4. The Scripts section contains one or more `### <name>` subsections, each defining one callable script
5. Within a script subsection, optional `key: value` lines between the `### <name>` heading and the first code fence are inline metadata (`kind`, `runtime`, `schema`)
6. The first code block following the inline metadata is the script body
7. Inline metadata defaults: `kind: tool`, `runtime` derived from code fence language
8. The first code block in the Database section (if any) contains the schema
9. The Actions section is parsed into a `map[string]string` of `### <Name>` subsections to their content
10. Unknown sections are preserved but not interpreted
11. Markdown formatting within sections is preserved verbatim
12. Script names within one cap must be unique

## Validation

A parser must reject a CAP.md as invalid if any of the following hold. Errors are fatal — implementations must not auto-correct or fall back to defaults.

| Condition | Error |
|-----------|-------|
| Frontmatter `name` field missing or empty | required field missing |
| Frontmatter `name` does not match the parent directory name | name/directory mismatch |
| `## Scripts` section missing | required section missing |
| `## Scripts` section contains zero `### <name>` subsections | empty Scripts |
| Two `### <name>` subsections share the same name within one cap | duplicate script name |
| `### <name>` subsection contains zero code fences | missing script body |
| `### <name>` subsection contains more than one code fence | ambiguous script body |
| Inline metadata contains an unknown key | unknown metadata key |
| Inline metadata `kind` is not `tool` or `handler` | invalid kind value |
| Inline metadata `runtime` is not `repl` or `bash` | invalid runtime value |
| Code fence language conflicts with explicit `runtime` value | runtime mismatch |
| `kind: tool` with `runtime: bash` and no `schema` field | missing schema for bash tool |
| `kind: handler` with a `schema` field | invalid schema on handler |

A parser may accept these without error (they are warnings or silently preserved):

- Unknown sections at the `##` level (preserved verbatim, ignored semantically)
- Markdown content between sections (preserved as prose)
- Trailing whitespace and blank lines

## Migration from v1.0

A v1.0 single-script CAP.md migrates to v1.1 by wrapping the existing `## Script` block in a `## Scripts` / `### <cap_name>` subsection:

**v1.0:**

````markdown
---
name: reddit_fetch
runtime: repl
schema: {"type":"object","properties":{"url":{"type":"string"}}}
---

## Purpose
Fetch a Reddit URL.

## Script
```javascript
async ({ url }) => { ... }
```
````

**v1.1:**

````markdown
---
name: reddit_fetch
---

## Purpose
Fetch a Reddit URL.

## Scripts

### reddit_fetch
kind: tool

```javascript
async ({ url }) => { ... }
```
````

The cap-level `runtime` and `schema` fields are removed (now per-script and derived). The script subsection name conventionally matches the cap name. There is no parser fallback for the v1.0 form: a v1.1 parser rejects v1.0 files with an "## Scripts section missing" error.



- **Tests** — belong in the script itself or in a separate test harness
- **Dependencies** — caps use the runtime's built-in utilities; if more is needed, use a classic plugin/skill
- **Permissions** — derived from `runtime` and script content at parse time
- **Author** — source attribution is handled by runtime metadata, not the cap file
