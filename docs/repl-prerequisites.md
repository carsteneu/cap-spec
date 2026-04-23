# REPL Prerequisites

A CAP.md handler runs inside a JavaScript REPL VM. This document lists what must be available in the execution scope for capabilities to work.

## Runtime Builtins

These are always available, regardless of provider. A capability can use them directly without adapters.

| Builtin | Signature | Purpose |
|---------|-----------|---------|
| `sh(cmd, ms?)` | `string → string` | Shell execution, returns stdout+stderr |
| `shQuote(s)` | `string → string` | Shell-escape a string for safe interpolation |
| `haiku(prompt, schema?)` | `string → any` | Single-turn LLM sampling (fast model) |
| `log(...)` | `...any → void` | Console output (alias for `console.log`) |
| `str(obj)` | `any → string` | JSON.stringify shorthand |
| `cat(path, off?, lim?)` | `string → string` | Read file content |
| `put(path, content)` | `string, string → void` | Write file content |
| `gl(pat, path?)` | `string → string[]` | Glob for file paths |
| `rg(pat, path?, opts?)` | `string → string` | Ripgrep search |
| `rgf(pat, path?, glob?)` | `string → string[]` | Ripgrep file list |

## Adapter Primitives

These are injected by the provider's adapter layer at activation time. A capability script uses generic names; the adapter maps them to provider-specific implementations.

| Primitive | Purpose | Injected by |
|-----------|---------|-------------|
| `store(...)` | Persistence (tables, blobs) | `GenerateAdapterJS()` or direct mapping |
| `web(...)` | Network I/O (fetch, search) | `GenerateAdapterJS()` dispatcher |
| `file(...)` | Filesystem I/O (read, write, glob) | `GenerateAdapterJS()` dispatcher |

The adapter JS is prepended to the handler code only when the handler actually uses generic adapter calls. Bash-only capabilities and handlers that call provider-specific functions directly skip the adapter prefix.

## Detection: `UsesGenericAdapters(script)`

Before prepending adapter JS, the system checks whether the handler script contains callsites for any generic primitive. This uses word-boundary matching to avoid false positives (e.g., `store(` inside `mcp__yesmem__cap_store(` is not a match).

## Tool Registration

Capabilities are registered in the REPL via `registerTool(name, description, schema, handler)`. This makes the capability callable as a native tool in subsequent turns — the proxy injects the tool schema into the API request, so the model sees it alongside built-in tools.

## Provider Contract

A minimal provider must supply:
1. The runtime builtins listed above (or equivalent)
2. An adapter mapping for the 3 primitives (`store`, `web`, `file`)
3. A `registerTool()` function that makes handlers callable as tools
