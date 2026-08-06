---
name: coderabbit
description: >-
  CodeRabbit CLI local code reviews and .coderabbit.yaml configuration.
  Run AI reviews on uncommitted/committed changes before pushing or opening
  a PR, parse --agent findings, iterate fix-and-verify loops within hourly
  rate limits, and tune repo config for low-noise high-signal reviews.
  Use when reviewing local changes pre-push/pre-PR, installing or driving
  the coderabbit/cr CLI, creating or tuning .coderabbit.yaml, or reducing
  CodeRabbit review noise. Not for PR-side review loops and thread
  handling (git-pr), commits (git-commit), or CI status (git-ci)
---

# CodeRabbit CLI and Configuration

**Local-first AI code review: catch issues before they reach the PR.** The CodeRabbit CLI (`coderabbit`, alias `cr`) reviews working-tree or branch changes locally, so the PR-side review becomes confirmation instead of iteration -- this saves billable PR review rounds (both CodeRabbit's own quota and any Copilot credits). Covers the CLI surface and `.coderabbit.yaml` tuning. PR-side mechanics (threads, re-requests, the bot loop) live in the `git-pr` skill.

## When to Use

- **Reviewing local changes** -- "review my changes", "run coderabbit", pre-commit/pre-push/pre-PR checks
- **Driving a review-fix loop** -- run review, fix valid findings, re-run to verify
- **Configuring CodeRabbit** -- create or tune `.coderabbit.yaml`, reduce review noise, disable redundant linters
- **Checking limits** -- rate limits per plan, review budgeting
- **Setting up the CLI** -- install, auth, headless/CI usage

## Critical Rules

1. **Reviews upload code to CodeRabbit's service.** A review sends the diff (and context) to CodeRabbit. On a repo with sensitive/unpublished code, confirm the user is OK with that before the first run.
2. **Reviews consume a per-hour quota** (Free: 3/hour CLI reviews). Scope deliberately (`--type`, `--base`) and use `coderabbit review findings` to replay the last result without spending a review.
3. **Use `--agent` output when driving fixes programmatically**, `--plain` when showing results to a human.
4. **Validate findings before fixing** -- same rule as PR reviews: judge each finding on its merits; never blind-fix to silence the tool.
5. **CodeRabbit is optional -- check the Code Review Policy first** (repo AGENTS.md/CLAUDE.md, falling back to the user's global agent instructions; see `git-pr` skill) for the preferred reviewer and checkpoints. Never install, authenticate, or run it on a project whose policy or user hasn't opted in.

---

## Setup

```bash
curl -fsSL https://cli.coderabbit.ai/install.sh | sh   # or: brew install coderabbit
coderabbit auth login        # browser OAuth
coderabbit auth status       # verify
coderabbit doctor            # diagnose runtime/auth/connectivity issues
```

**No paid plan required**: the Free plan includes CLI reviews (3/hour) after `coderabbit auth login` with a free account -- paid plans add org context, learnings, and higher limits. Headless/CI: `CODERABBIT_API_KEY` env var with an **Agentic API key** (requires the usage-based add-on on paid plans), or `--api-key` per call. `coderabbit update` self-updates.

The CLI must run inside a git repository. `cr` is a shorthand alias for `coderabbit`.

## Review Scopes and Checkpoints

| Checkpoint | Command | Reviews |
|------------|---------|---------|
| Before commit | `coderabbit review --type uncommitted --plain` | Staged + unstaged working-tree changes |
| Before push | `coderabbit review --type committed --plain` | Local commits not in the base branch |
| Before PR (full branch) | `coderabbit review --type all --base main --plain` | Working tree + branch commits vs base |
| Subdirectory only | `coderabbit review --type all --dir packages/api --plain` | Changes under a path |
| Vs specific commit | `coderabbit review --type committed --base-commit {sha} --plain` | Changes since a commit |

Defaults: `--type all`, base = repository default branch. `--light` runs a faster, lighter review policy for quick local iteration.

## Output Modes

- `--plain` -- detailed human-readable feedback with fix suggestions (default mode is plain, non-interactive)
- `--agent` -- JSON-lines: one object per line. Finding objects carry `type: "finding"`, `severity` (`critical|major|minor|trivial|info`), `fileName`, `comment`, `suggestions`, and `codegenInstructions` (written for coding agents -- follow them when fixing). Heartbeat events appear during long reviews; a final `complete` event carries `status` (`"review_skipped"` with `findings: 0` when the scope has no changes).
- `coderabbit review findings` -- replay the cached findings from the last review with no new analysis and **no quota cost**. Use between fix iterations; only re-run a real review to verify at the end.
- `coderabbit stats` -- review statistics.

Note: `--prompt-only` is a deprecated alias of `--agent`; generate `--agent`.

## The Local Review-Fix Loop

1. Run `coderabbit review --type {scope} --agent` (background it -- reviews take minutes).
2. Parse findings; triage by `severity`. Address `critical` and `major` first.
3. **Validate each finding** against the codebase (conventions, actual behavior, project docs). Fix valid ones per `codegenInstructions`; note invalid ones with a one-line rationale for the user.
4. Re-run the same review command to verify fixes. Stop when no valid `critical`/`major` findings remain, or the hourly bucket is exhausted (the CLI reports the limit -- wait or stop, never hammer).
5. Then push / create the PR; the PR-side review (if any) should come back clean or near-clean.

Commit fixes by what they change, never by what prompted them -- `fix: validate empty page cursor`, not `fix: coderabbit fixes` or `fix: review round 2` (see the `git-commit` skill). Fixes must not add code comments that restate what the code already reads.

Two passes (review, fix, verify) is the normal shape. More than three passes means findings are being treated as noise -- re-evaluate validity or tune config (see `references/configuration.md`).

## Rate Limits (Per Developer, Per Hour)

| Plan | CLI reviews | PR reviews | Files/review |
|------|-------------|------------|--------------|
| Free | 3 | 1 (summary only) | 150 |
| Pro | 5 | 5 | 300 |
| Pro+ | 10 | 10 | 300 |
| Enterprise | 12 | 12 | 300 |

The Lite plan was retired (June 2026); Free / Pro / Pro+ / Enterprise are current. Beyond the hourly allowance, the usage-based add-on bills $0.25 per reviewed file (Pro and up). Open-source public repos get free reviews with popularity-based limits.

## Configuration

`.coderabbit.yaml` at the repo root governs both PR-side and CLI reviews (the CLI also accepts `-c <file>` for extra instruction files, e.g. `CLAUDE.md`). For typed, well-linted projects the goal is high-level findings only: `profile`, `tone_instructions`, disabled CI-redundant linters, `path_filters` for generated files.

> **Reference**: See `references/configuration.md` for the full schema highlights, config precedence, the low-noise template for typed/linted projects, and PR-side commands (`@coderabbitai review` vs `full review`, pause/resume, config dump).

## Key Gotchas

1. **`--type committed` needs commits, `uncommitted` needs a dirty tree** -- reviewing the wrong scope silently reviews nothing (`review_skipped`); match the scope to the checkpoint.
2. **`coderabbit review findings` replays, it does not re-review** -- after fixing, cached findings still show; only a fresh review verifies fixes.
3. **The quota is per-hour, not per-day** -- a "limit reached" message means wait for the window, not stop for the day. Plan verify runs so the final pass fits the bucket.
4. **CLI reviews and PR reviews draw from separate buckets** -- burning CLI reviews locally does not reduce the PR-side allowance, which is the point of the local-first flow.
5. **Org context needs matching access** -- on a repo not linked to your CodeRabbit org, reviews run in limited free mode (no learnings/org context); results differ from PR-side reviews on the same code.
6. **`--agent` emits JSON lines, not a JSON document** -- parse line-by-line; do not `JSON.parse` the whole output.

> **Reference**: See `references/configuration.md` for `.coderabbit.yaml` tuning and PR commands
> **Reference**: See `references/allowlist.md` for auto-approval patterns
> **Reference**: PR-side review loops, thread handling: `git-pr` skill (`references/bot-review-loop.md`)
