# Agent Skills

A collection of agent skills for [OpenCode](https://opencode.ai), [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and other AI agents, installable via [skills.sh](https://skills.sh).

## Available Skills

| Skill | Description | Install |
|---|---|---|
| `bun-cli` | Bun CLI: package management, scripts, testing, bundling, compilation | `bunx skills add dmythro/agent-skills --skill bun-cli` <br> `npx skills add dmythro/agent-skills --skill bun-cli` |
| `bun-api` | Bun runtime API: file I/O, shell, SQLite, hashing, compression, utilities | `bunx skills add dmythro/agent-skills --skill bun-api` <br> `npx skills add dmythro/agent-skills --skill bun-api` |
| `git-commit` | Conventional Commits format for all git commits and PR/MR titles | `bunx skills add dmythro/agent-skills --skill git-commit` <br> `npx skills add dmythro/agent-skills --skill git-commit` |
| `git-pr` | PR and MR workflows for GitHub (gh) and GitLab (glab): creation, review, comments, merging | `bunx skills add dmythro/agent-skills --skill git-pr` <br> `npx skills add dmythro/agent-skills --skill git-pr` |
| `git-ci` | CI/CD status queries for GitHub Actions (gh) and GitLab CI (glab) | `bunx skills add dmythro/agent-skills --skill git-ci` <br> `npx skills add dmythro/agent-skills --skill git-ci` |

## Install

Install individual skills:

```bash
bunx skills add dmythro/agent-skills --skill bun-cli
bunx skills add dmythro/agent-skills --skill bun-api
bunx skills add dmythro/agent-skills --skill git-commit
bunx skills add dmythro/agent-skills --skill git-pr
bunx skills add dmythro/agent-skills --skill git-ci

# or with npx
npx skills add dmythro/agent-skills --skill bun-cli
npx skills add dmythro/agent-skills --skill bun-api
npx skills add dmythro/agent-skills --skill git-commit
npx skills add dmythro/agent-skills --skill git-pr
npx skills add dmythro/agent-skills --skill git-ci
```

Or install all skills at once:

```bash
bunx skills add dmythro/agent-skills
# or
npx skills add dmythro/agent-skills
```

## Update

```bash
bunx skills update
# or
npx skills update
```

## Recommended Skills

Other useful skills from the community:

| Skill | Description | Install |
|---|---|---|
| `gh-cli` | GitHub CLI: comprehensive command reference for repos, PRs, issues, Actions, releases | `bunx skills add github/awesome-copilot --skill gh-cli` <br> `npx skills add github/awesome-copilot --skill gh-cli` |
| `next-best-practices` | Next.js: RSC, data patterns, routing, metadata, error handling, optimization | `bunx skills add vercel-labs/next-skills --skill next-best-practices` <br> `npx skills add vercel-labs/next-skills --skill next-best-practices` |
| `elysiajs` | Elysia: type-safe Bun web framework -- routing, validation, auth, WebSocket | `bunx skills add elysiajs/skills --skill elysiajs` <br> `npx skills add elysiajs/skills --skill elysiajs` |

## Agent Allowlist

Skills with CLI commands include a `references/allowlist.md` with tiered auto-approval patterns for Claude Code `settings.json` or OpenCode config -- read-only commands safe to auto-approve:

- [`bun-cli/references/allowlist.md`](skills/bun-cli/references/allowlist.md) -- `bun info`, `bun pm`, `bun run`, `bun test`, etc.
- [`git-pr/references/allowlist.md`](skills/git-pr/references/allowlist.md) -- `gh pr view`, `gh issue list`, `glab mr view`, `glab mr list`, etc.
- [`git-ci/references/allowlist.md`](skills/git-ci/references/allowlist.md) -- `gh pr checks`, `gh run list`, `glab ci status`, `glab ci list`, etc.

## License

MIT
