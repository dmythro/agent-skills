---
name: gh-cli-flow
description: Opinionated GitHub workflows for PR review, comment handling, CI
  queries, and tool allowlists. Pairs with gh-cli for command reference
---

# GitHub CLI Coding Flow

**Opinionated workflow layer for GitHub CLI.** This skill defines _how_ to work with PRs, reviews, and CI -- not what commands exist. For full command reference, install the companion skill:

```bash
npx skills add github/awesome-copilot --skill gh-cli
```

## When to Use

- Deciding the right sequence of `gh` commands for a PR workflow
- Handling PR review comments (fetch, evaluate, fix, reply, resolve)
- Posting line-specific comments on PRs (requires `gh api`)
- Querying review state, merge readiness, or pending reviewers
- Looking up read-only `gh api` endpoints not covered by subcommands
- Configuring tool allowlists for safe auto-approval of read-only commands

## Critical Rule

**Prefer `gh` subcommands over raw `gh api` when a subcommand exists.** Subcommands handle pagination, error formatting, and repo detection automatically. Only use `gh api` for operations not covered by subcommands (line-specific PR comments, review thread resolution, GraphQL queries).

---

## Read-Only vs Write Classification

All `gh` commands fall into two categories:

- **Read-only** (safe to auto-approve): `view`, `list`, `status`, `diff`, `checks`, `search`, `workflow list`
- **Write** (require user approval): `create`, `edit`, `merge`, `close`, `reopen`, `comment`, `review`, `run rerun`, `run cancel`, `workflow run`

See `references/allowlist.md` for copy-paste ready auto-approval patterns.

---

## PR Review Comment Handling

When asked to "check PR comments", "review comments", or "address feedback":

> **Reference**: See `references/pr-comment-workflow.md` for the full opinionated workflow -- fetch unresolved threads, evaluate each comment critically, fix/reply/resolve.

Quick summary:
1. Fetch unresolved review threads (GraphQL query)
2. For each thread: read the code context, check if already addressed
3. Decide: fix code + reply + resolve, or reply with evidence + resolve, or leave for discussion
4. Don't blindly agree -- validate comments against actual code and conventions

---

## Common PR Workflows

**Create PR from current branch:**
```bash
gh pr create --draft --fill
```

**Quick review cycle:**
```bash
gh pr checks 123 --watch           # Wait for CI
gh pr diff 123 --stat              # Review scope
gh pr view 123 --json reviews      # Check review state
gh pr merge 123 --squash --delete-branch
```

**Find PRs needing review:**
```bash
gh pr list --search "review-requested:@me"
gh pr list --search "review:required -review:approved draft:false"
```

**Check workflow runs for a specific workflow:**
```bash
gh workflow list --json name,id --jq '.[] | "\(.id) \(.name)"'
gh run list --workflow ci.yml --limit 5 --json status,conclusion,headBranch
```

---

## Line-Specific PR Comments

Line comments require `gh api` -- no subcommand exists for this.

> **Reference**: See `references/line-comments.md` for full `gh api` patterns: single-line comments, multi-line ranges, replies to threads, edit/delete.

Quick reference:
```bash
# Single-line comment on latest commit
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="Comment" -f path="src/file.ts" -F line=42 -f side=RIGHT

# Reply to a review comment thread
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Reply text"
```

---

## Review State & CI Queries

> **Reference**: See `references/review-queries.md` for advanced queries: full review state summary, pending reviewer detection, merge readiness assessment, PR metadata extraction.

Quick patterns:
```bash
# Who approved / requested changes
gh pr view 123 --json reviews --jq '.reviews | group_by(.author.login) | map({user: .[0].author.login, latest: .[-1].state})'

# Files changed with additions/deletions
gh pr view 123 --json files --jq '.files[] | "\(.additions)+/\(.deletions)- \(.path)"'
```

---

## Read-Only gh api Endpoints

When a subcommand doesn't exist, use `gh api` for read-only access.

> **Reference**: See `references/api-readonly.md` for the full list of read-only REST and GraphQL endpoints (PR data, issue timelines, review threads).

---

## Allowlist Reference

> **Reference**: See `references/allowlist.md` for copy-paste ready `Bash(command:*)` patterns covering all read-only operations -- safe to auto-approve in Claude Code `settings.json` or OpenCode config.
