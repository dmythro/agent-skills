# Agent Skills

A collection of agent skills for [OpenCode](https://opencode.ai), [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and other AI agents, installable via [skills.sh](https://skills.sh).

## Available Skills

| Skill | Description | Install |
|---|---|---|
| `bun-cli` | Bun CLI: package management, scripts, testing, bundling, compilation | `npx skills add dmythro/agent-skills --skill bun-cli` |
| `bun-api` | Bun runtime API: file I/O, shell, SQLite, hashing, compression, utilities | `npx skills add dmythro/agent-skills --skill bun-api` |

## Recommended Skills

Other useful skills from the community:

| Skill | Description | Install |
|---|---|---|
| `gh-cli` | GitHub CLI: repos, issues, PRs, Actions, releases, and all gh operations | `npx skills add github/awesome-copilot --skill gh-cli` |
| `next-best-practices` | Next.js: RSC, data patterns, routing, metadata, error handling, optimization | `npx skills add vercel-labs/next-skills --skill next-best-practices` |
| `elysiajs` | Elysia: type-safe Bun web framework — routing, validation, auth, WebSocket | `npx skills add elysiajs/skills --skill elysiajs` |

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
