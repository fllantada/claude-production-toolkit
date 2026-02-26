# Claude Code Expert — Memory Index

> Accumulated knowledge index. Read this file first to know what's available.

## Reference files

| Date | File | Topic | Source |
|------|------|-------|--------|
| 2026-02-26 | `2026-02-26-gsd-patterns.md` | GSD patterns: agents, commands, hooks, state | github.com/gsd-build/get-shit-done (~20k stars) |
| 2026-02-26 | `2026-02-26-hook-format-reference.md` | Claude Code hook JSON format (input/output per tool) | Official docs + GSD cross-verification |

## Verified gotchas

1. **jq `//` treats `0` as falsy** — In bash hooks using `.exit_code // "1"`, an exit code of 0 gets lost. Use Node.js or `if/then/else` in jq. (2026-02-26)

## Reference repos

| Repo | Stars | Last analysis | Specialty |
|------|-------|---------------|-----------|
| `gsd-build/get-shit-done` | ~20.5k | 2026-02-26 | Hooks Node.js, agents, commands, state |

## Pending investigation

- Exact `tool_response` format for tools: Grep, Glob, Edit, Write (only verified Bash and Read)
- How GSD handles skills vs commands (exact difference in Claude Code)
- `Notification` and `ConfigChange` hook types (poorly documented)
- Other successful Claude Code tooling repos
