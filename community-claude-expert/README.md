# Community Claude Expert

A Claude Code expert agent trained on community repos instead of official documentation.

## The idea

Official docs for fast-evolving tools like Claude Code lag behind real-world usage. This agent learns from **battle-tested open-source repos** — their hook patterns, agent structures, state management — and applies that knowledge to audit and improve your own Claude Code setup.

Read the full story: [Community-Trained Agent: Learning from a Repo, Not from Docs](https://dev-fran.com/en/blog/community-trained-agent)

## What's included

```
community-claude-expert/
├── README.md                           # This file
├── agents/
│   └── claude-code-expert.md           # The agent definition
└── agent-memory/
    └── claude-code-expert/
        ├── INDEX.md                    # Memory index
        ├── 2026-02-26-gsd-patterns.md  # Patterns from GSD repo
        └── 2026-02-26-hook-format-reference.md  # Hook JSON formats
```

## Installation

1. Copy `agents/claude-code-expert.md` to your project's `.claude/agents/`
2. Copy the `agent-memory/claude-code-expert/` folder to `.claude/agent-memory/`
3. The agent is now available as a subagent via the Task tool

```bash
# From your project root:
cp community-claude-expert/agents/claude-code-expert.md .claude/agents/
cp -r community-claude-expert/agent-memory/claude-code-expert .claude/agent-memory/
```

## How it works

1. You ask a question about Claude Code configuration
2. The agent reads its **persistent memory** (timestamped discovery files)
3. If the memory doesn't have the answer, it investigates **reference repos** via GitHub API
4. It returns concrete code, not theory
5. New discoveries get saved to memory for next time

## Reference repos

The agent currently learns from:

| Repo | Stars | What it teaches |
|------|-------|----------------|
| [gsd-build/get-shit-done](https://github.com/gsd-build/get-shit-done) | ~20k | Hook patterns (Node.js), agent XML structure, command orchestration, state management |

## Adding your own reference repos

Edit the agent definition and add repos to the reference table. The agent will investigate them when its memory is insufficient.

## License

MIT
