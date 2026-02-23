# Agent Skills

A collection of agent skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and other AI agents, installable via [skills.sh](https://skills.sh).

## Available Skills

| Skill | Description | Install |
|---|---|---|
| `bun-cli` | Bun CLI: package management, scripts, testing, bundling, compilation | `npx skills add dmythro/agent-skills --skill bun-cli` |
| `bun-api` | Bun runtime API: file I/O, shell, SQLite, hashing, compression, utilities | `npx skills add dmythro/agent-skills --skill bun-api` |

## Install

Install individual skills:

```bash
npx skills add dmythro/agent-skills --skill bun-cli
npx skills add dmythro/agent-skills --skill bun-api
```

Or install all skills at once:

```bash
npx skills add dmythro/agent-skills
```

## Update

```bash
npx skills update
```

## License

MIT
