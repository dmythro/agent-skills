---
name: gh-pr
description: GitHub CLI reference for pull requests, code review, issues,
  and Actions. Covers PR creation/review/merge, line-specific comments,
  issue management, CI status, and workflow monitoring.
---

# GitHub PRs, Issues & Actions

Use `gh` subcommands for all GitHub operations. Subcommands handle pagination, error formatting, and repo detection automatically.

## When to Use

- Working with GitHub pull requests (create, review, merge, comment)
- Managing issues (create, view, comment, close, link to PRs)
- Checking CI/Actions status (runs, checks, workflow monitoring)

## Critical Rule

**Prefer `gh` subcommands over raw `gh api` when a subcommand exists.** Subcommands handle pagination, error formatting, and repo detection automatically. Only use `gh api` for operations not covered by subcommands (line-specific PR comments, review thread resolution, GraphQL queries).

---

## Read-Only Operations

These commands are safe to run without user approval. They only read data and have no side effects.

### Pull Requests

```bash
# View a PR (default: human-readable summary)
gh pr view 123
gh pr view 123 --json title,state,body,author,reviews,comments
gh pr view 123 --json reviews --jq '.reviews[].state'

# List PRs
gh pr list
gh pr list --state open --author @me
gh pr list --label bug --limit 50
gh pr list --search "review:required draft:false"
gh pr list --json number,title,state,updatedAt

# Current branch PR status
gh pr status

# Diff
gh pr diff 123                   # Full diff
gh pr diff 123 --name-only       # Changed file names
gh pr diff 123 --stat            # Diffstat summary
gh pr diff 123 --patch           # Patch format

# CI checks on a PR
gh pr checks 123
gh pr checks 123 --watch --interval 5   # Poll until done
gh pr checks 123 --json name,state,conclusion
```

### Issues

```bash
# View an issue
gh issue view 123
gh issue view 123 --json title,state,body,author,labels,comments
gh issue view 123 --json comments --jq '.comments[].body'

# List issues
gh issue list
gh issue list --state open --label bug
gh issue list --assignee @me
gh issue list --search "is:open label:priority"
gh issue list --json number,title,state,labels

# Issues assigned to you / created by you
gh issue status
```

### Actions & CI

```bash
# View a workflow run
gh run view 12345
gh run view 12345 --log               # Full log output
gh run view 12345 --log-failed        # Only failed step logs
gh run view 12345 --json status,conclusion,jobs

# List workflow runs
gh run list
gh run list --workflow ci.yml
gh run list --status failure --limit 10
gh run list --branch main
gh run list --json databaseId,status,conclusion,name,headBranch
```

### Search

```bash
# Search PRs across repos
gh search prs "query" --owner org --repo repo --state open
gh search prs "review:approved" --merged --limit 20

# Search issues
gh search issues "query" --label bug --state open
gh search issues "is:open assignee:@me"
```

### JSON Output Patterns

Extract structured data with `--json` and `--jq`:

```bash
# PR titles and URLs
gh pr list --json number,title,url --jq '.[] | "\(.number) \(.title)"'

# PR review state summary
gh pr view 123 --json reviews --jq '[.reviews[] | .state] | group_by(.) | map({(.[0]): length}) | add'

# Failing check names
gh pr checks 123 --json name,conclusion --jq '.[] | select(.conclusion == "FAILURE") | .name'

# Issue labels as comma-separated list
gh issue view 123 --json labels --jq '[.labels[].name] | join(", ")'

# Run IDs for failed runs
gh run list --status failure --json databaseId --jq '.[].databaseId'
```

---

## Write Operations

These commands modify state and require user approval.

### PR Create & Edit

```bash
# Create PR (interactive by default)
gh pr create
gh pr create --draft --title "feat: add X" --body "Description"
gh pr create --reviewer user1,user2 --assignee @me
gh pr create --label enhancement --milestone v2.0
gh pr create --base main --head feature-branch
gh pr create --fill                # Title/body from commits

# Edit PR metadata
gh pr edit 123 --title "new title"
gh pr edit 123 --body "new body"
gh pr edit 123 --add-label bug --remove-label triage
gh pr edit 123 --add-reviewer user1
gh pr edit 123 --add-assignee @me
gh pr edit 123 --milestone v2.0
```

### PR Review & Comment

```bash
# Submit a review
gh pr review 123 --approve
gh pr review 123 --approve --body "LGTM"
gh pr review 123 --request-changes --body "Please fix X"
gh pr review 123 --comment --body "Some notes"

# Dismiss a review (requires review ID)
# Use gh api for this — see references/review-queries.md

# Add a general comment to a PR
gh pr comment 123 --body "Comment text"
gh pr comment 123 --body-file comment.md

# Edit / delete comments
gh pr comment 123 --edit-last --body "Updated"
```

### PR Merge & Lifecycle

```bash
# Merge strategies
gh pr merge 123 --squash --delete-branch
gh pr merge 123 --rebase
gh pr merge 123 --merge
gh pr merge 123 --auto --squash    # Auto-merge when checks pass

# Lifecycle
gh pr close 123
gh pr reopen 123
gh pr ready 123                    # Mark as ready for review

# Update branch (rebase or merge base into PR)
gh pr update-branch 123
gh pr update-branch 123 --rebase

# Checkout PR locally
gh pr checkout 123
```

### Issue Create & Edit

```bash
# Create issue
gh issue create --title "Bug: X" --body "Description"
gh issue create --label bug --assignee @me
gh issue create --template bug_report.md

# Edit issue
gh issue edit 123 --title "updated title"
gh issue edit 123 --add-label priority --remove-label triage
gh issue edit 123 --add-assignee user1

# Comment
gh issue comment 123 --body "Comment text"

# Close / reopen
gh issue close 123
gh issue close 123 --reason "not planned"
gh issue reopen 123

# Pin / lock
gh issue pin 123
gh issue unpin 123
gh issue lock 123 --reason spam
```

### Actions

```bash
# Rerun a failed workflow
gh run rerun 12345
gh run rerun 12345 --failed        # Only failed jobs

# Cancel a running workflow
gh run cancel 12345

# Trigger a workflow
gh workflow run ci.yml
gh workflow run deploy.yml --ref main -f environment=staging
```

---

## Workflows

### PR Review Comment Handling

When asked to "check PR comments", "review comments", or "address feedback":

> **Reference**: See `references/pr-comment-workflow.md` for the full opinionated workflow — fetch unresolved threads, evaluate each comment critically, fix/reply/resolve.

Quick summary:
1. Fetch unresolved review threads (GraphQL query)
2. For each thread: read the code context, check if already addressed
3. Decide: fix code + reply + resolve, or reply with evidence + resolve, or leave for discussion
4. Don't blindly agree — validate comments against actual code and conventions

### Common PR Workflows

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

---

## Line-Specific PR Comments

Line comments require `gh api` — no subcommand exists for this.

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

When a subcommand doesn't exist, these `gh api` calls are GET-only and safe:

> **Reference**: See `references/api-readonly.md` for the full list of read-only REST and GraphQL endpoints.

```bash
# PR data
gh api repos/{owner}/{repo}/pulls/{pr}
gh api repos/{owner}/{repo}/pulls/{pr}/comments
gh api repos/{owner}/{repo}/pulls/{pr}/reviews
gh api repos/{owner}/{repo}/pulls/{pr}/files

# Issue data
gh api repos/{owner}/{repo}/issues/{issue}/comments
gh api repos/{owner}/{repo}/issues/{issue}/timeline

# GraphQL (POST method, but read-only queries)
gh api graphql -f query='{ repository(owner:"org", name:"repo") { pullRequest(number:123) { reviewThreads(first:50) { nodes { isResolved comments(first:10) { nodes { body author { login } } } } } } } }'
```

---

## Allowlist Reference

> **Reference**: See `references/allowlist.md` for copy-paste ready `Bash(command:*)` patterns covering all read-only operations in this skill.
