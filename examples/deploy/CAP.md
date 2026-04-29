---
name: deploy
description: Run `make deploy` in the current yesmem worktree — builds binary, atomic-replaces ~/.local/bin/yesmem, restarts daemon & proxy services.
version: 3
tags:
  - ops
  - deploy
tested: true
---

## Purpose

Single-command deploy for the yesmem binary. Builds from the current worktree,
atomically replaces the installed binary, and restarts all services.

Use when you've finished a feature or fix and want to deploy it.

Optional parameter `worktree` overrides which directory to build from
(defaults to the current working directory).

## Scripts

### deploy
kind: tool
schema: {"type":"object"}

```bash
cd "${WORKTREE:-.}" && make deploy
```

## Database

