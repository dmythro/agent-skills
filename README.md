# Agent Skills

A collection of agent skills for [OpenCode](https://opencode.ai), [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and other AI agents, installable via [skills.sh](https://skills.sh).

## Available Skills

| Skill | Description | Install |
|---|---|---|
| `bun-cli` | Bun CLI: package management, scripts, testing, bundling, compilation | `npx skills add dmythro/agent-skills --skill bun-cli` |
| `bun-api` | Bun runtime API: file I/O, shell, SQLite, hashing, compression, utilities | `npx skills add dmythro/agent-skills --skill bun-api` |
| `gh-cli-flow` | GitHub CLI: pull requests, code review, issues, Actions, workflows, search, and labels | `npx skills add dmythro/agent-skills --skill gh-cli-flow` |

## Install

Install individual skills:

```bash
npx skills add dmythro/agent-skills --skill bun-cli
npx skills add dmythro/agent-skills --skill bun-api
npx skills add dmythro/agent-skills --skill gh-cli-flow
```

Or install all skills at once:

```bash
npx skills add dmythro/agent-skills
```

## Update

```bash
npx skills update
```

## Recommended Skills

Other useful skills from the community:

| Skill | Description | Install |
|---|---|---|
| `next-best-practices` | Next.js: RSC, data patterns, routing, metadata, error handling, optimization | `npx skills add vercel-labs/next-skills --skill next-best-practices` |
| `elysiajs` | Elysia: type-safe Bun web framework — routing, validation, auth, WebSocket | `npx skills add elysiajs/skills --skill elysiajs` |

## Agent Allowlist

Skills with CLI commands include a `references/allowlist.md` with suggested `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config — read-only commands safe to auto-approve:

- [`bun-cli/references/allowlist.md`](skills/bun-cli/references/allowlist.md) — `bun info`, `bun pm`, `bun run`, `bun test`, etc.
- [`gh-cli-flow/references/allowlist.md`](skills/gh-cli-flow/references/allowlist.md) — `gh pr view`, `gh issue list`, `gh run view`, `gh workflow list`, `gh search code`, etc.

## License

MIT
