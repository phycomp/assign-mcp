# assign-mcp

A skill for assigning work to agents via OpenMausBot MCP — creating role-based bots, grouping them into channels, sending kickoff messages, and tracking task progress.

## What it does

- Discover available bots and channels
- Create tasks for bots or channels
- Send kickoff messages with idempotency support
- Track progress (messages, busy state, activity)
- Interrupt, switch, rename, and search tasks
- Provides gotchas and anti-patterns

## Install

```bash
npx skills add phycomp/assign-mcp@assign-mcp -g -y
```

## Usage

Trigger phrases: "assign task to bot", "OpenMausBot", "create channel", "kickoff team", "assign MCP task", or when deploying AI teammates for parallel work.

## Contents

- `SKILL.md` — skill instructions and workflow

## License

MIT
