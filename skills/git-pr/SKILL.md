---
name: git-pr
description: >-
  PR and MR workflows for GitHub (gh) and GitLab (glab). Creation, review
  comment handling, thread resolution, review state queries, and merging.
  Use when creating PRs/MRs, addressing review feedback, resolving threads, checking
  approvals, querying PR data, or configuring gh/glab read-only allowlists.
  Not for git commits (git-commit), CI/CD status (git-ci), or general git ops
---

# PR and MR Workflows

**Primary skill for pull request and merge request workflows across GitHub and GitLab.** Covers the full lifecycle: creation, review queries, comment handling, line-specific comments, and merging. All recipes use minimal field sets for token efficiency.

## Context Check (Do This First)

Before starting any PR workflow, detect the current state. This determines the right action:

```bash
# Check if a PR exists for the current branch (allowlisted, zero approval)
gh pr view --json number,state,reviewDecision,reviewRequests,title 2>/dev/null
```

| Result                                     | Next action                                                |
|--------------------------------------------|------------------------------------------------------------|
| PR exists, `CHANGES_REQUESTED`             | Fetch unresolved threads (see Review Comment Handling)      |
| PR exists, `REVIEW_REQUIRED` or has pending `reviewRequests` | Check review state or wait for reviewers  |
| PR exists, `APPROVED`                      | Check CI status or proceed with merge                      |
| PR exists, no review decision yet          | Check CI status, review state, or push more changes        |
| No PR for current branch                   | Create a PR (see PR/MR Creation)                           |

This avoids offering to create a PR when one already exists, and immediately surfaces pending review work. The `reviewDecision` field reliably indicates whether a reviewer has requested changes without needing to fetch individual threads.

## When to Use

- **Creating PRs or MRs** -- draft workflows, fill patterns, title format
- **Addressing PR/MR review feedback** -- "fix PR comments", "address review", "handle feedback", "resolve review threads"
- **Responding to code review** -- evaluating reviewer comments, replying, resolving threads
- **Checking review state** -- approvals, pending reviewers, review decisions
- **Querying PR/MR data** -- files changed, commits, labels, linked issues
- **Posting comments on PRs/MRs** -- line-specific comments, thread replies
- **Configuring tool allowlists** -- auto-approval patterns for read-only commands

## Critical Rules

1. **Prefer CLI subcommands over raw API calls.** Subcommands handle pagination, error formatting, and repo detection. Only use `gh api` / `glab api` for operations not covered by subcommands (line comments, thread resolution, GraphQL).
2. **Use `--json field1,field2` with `gh` to filter output.** This IS the efficiency mechanism -- no `--jq` needed for basic queries. Only request fields you actually need.
3. **`glab` has no `--json field1,field2` equivalent.** Use `-F json | jq '{fields}'` to filter output for token efficiency.
4. **Use commands exactly as shown in this skill.** The commands below are designed to match auto-approval allowlist patterns. Improvising flag order or adding unexpected flags may trigger permission prompts.

---

## Provider Detection

```bash
git remote get-url origin
```

| Remote URL contains                | Provider | CLI    | PR term |
|------------------------------------|----------|--------|---------|
| `github.com`                       | GitHub   | `gh`   | PR      |
| `gitlab.com` or self-hosted GitLab | GitLab   | `glab` | MR      |

If ambiguous or both present, ask the user.

---

## Read-Only vs Write Classification

- **Read-only** (safe to auto-approve): `view`, `list`, `status`, `diff`, `checks`, `search`
- **Write** (require user approval): `create`, `edit`, `merge`, `close`, `reopen`, `comment`, `review`, `approve`

> **Reference**: See `references/allowlist.md` for tiered auto-approval patterns.

---

## PR/MR Summary (Current Branch)

**GitHub:**
```bash
gh pr view --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName
```

**GitLab:**
```bash
glab mr view -F json | jq '{iid:.iid,title:.title,state:.state,draft:.draft,merge_status:.merge_status,target:.target_branch,source:.source_branch}'
```

## PR/MR Summary (By Number)

**GitHub:**
```bash
gh pr view {number} --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName
```

**GitLab:**
```bash
glab mr view {iid} -F json | jq '{iid:.iid,title:.title,state:.state,draft:.draft,merge_status:.merge_status,target:.target_branch,source:.source_branch}'
```

## Review State

> **Reference**: See `references/review-queries.md` for advanced review queries: per-reviewer state, approval checks, pending reviewers.

**GitHub:**
```bash
gh pr view --json reviews,reviewRequests,latestReviews
```

**GitLab:**
```bash
glab mr view -F json | jq '{upvotes:.upvotes,reviewers:[.reviewers[]?.username]}'
```

For detailed approval info (GitLab):
```bash
glab api projects/{project_id}/merge_requests/{iid}/approvals | jq '{approved:.approved,approvers:[.approved_by[]?.user.username]}'
```

## Files Changed

**GitHub:**
```bash
gh pr diff --name-only
```

**GitLab:**
```bash
glab mr diff
```

## File Stats

**GitHub:**
```bash
gh pr view --json files
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/changes | jq '[.changes[] | {path:.new_path,added:.diff | split("\n") | map(select(startswith("+"))) | length,removed:.diff | split("\n") | map(select(startswith("-"))) | length}]'
```

## List Open PRs/MRs

**GitHub:**
```bash
gh pr list --json number,title,author,reviewDecision,updatedAt
```

**GitLab:**
```bash
glab mr list -F json | jq '[.[] | {iid:.iid,title:.title,author:.author.username,updated:.updated_at}]'
```

## PR/MR Commits

**GitHub:**
```bash
gh pr view --json commits
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/commits | jq '[.[] | {sha:.short_id,title:.title}]'
```

## Search PRs/MRs

**GitHub:**
```bash
gh pr list --search "review-requested:@me" --json number,title,url
```

**GitLab:**
```bash
glab mr list --reviewer=@me -F json | jq '[.[] | {iid:.iid,title:.title,url:.web_url}]'
```

---

## Create PR/MR (Write -- Manual Approval)

| Action            | GitHub                                                 | GitLab                                                    |
|-------------------|--------------------------------------------------------|-----------------------------------------------------------|
| Create draft      | `gh pr create --draft --fill`                          | `glab mr create --draft --fill`                           |
| Create with title | `gh pr create --title "feat: ..." --body "..."`        | `glab mr create --title "feat: ..." --description "..."` |

## Merge (Write -- Manual Approval)

| Action       | GitHub                                | GitLab                                        |
|--------------|---------------------------------------|-----------------------------------------------|
| Squash merge | `gh pr merge --squash --delete-branch` | `glab mr merge --squash --remove-source-branch` |
| Rebase merge | `gh pr merge --rebase --delete-branch` | `glab mr merge --rebase --remove-source-branch` |

---

## Review Comment Handling

> **Reference**: See `references/pr-comment-workflow.md` for the full opinionated workflow with all command patterns and examples.

The workflow has two distinct phases -- never mix them:

**Phase 1: Analyze and Fix (local work, no GitHub API writes, zero approvals)**
1. Fetch all unresolved review threads in a single GraphQL query with inline `--jq` filter
2. For each thread: read the file at the referenced path+line, check if the comment is valid by researching the codebase (patterns, conventions, CLAUDE.md, git log)
3. Be critical -- validate each comment against actual code before accepting. Reviewers can be wrong.
4. Make all necessary code fixes
5. Commit and push the fixes

**Phase 2: Reply and Resolve (one batched command, one approval)**
6. Combine all replies and all resolves into a single `&&`-chained command
7. REST replies first, then a single GraphQL mutation with aliases to batch-resolve all handled threads
8. Leave "Needs discussion" threads unresolved

This ordering matters: pushing fixes first ensures reviewers see the changes when they read replies. Never reply to a comment claiming "Fixed" before the fix is actually pushed.

### Fetch Unresolved Threads (Zero Approvals)

**GitHub** -- one command with `$(...)` substitution. **Generate as a single line** and do NOT prepend variable assignments (`OWNER=...`, `REPO=...`) -- both break allowlist matching:

```bash
gh api graphql -f query="{ repository(owner: \"$(gh repo view --json owner --jq '.owner.login')\", name: \"$(gh repo view --json name --jq '.name')\") { pullRequest(number: $(gh pr view --json number --jq '.number')) { reviewThreads(first: 100) { nodes { id isResolved isOutdated path line startLine comments(first: 20) { nodes { id databaseId body author { login } } } } } } } }" --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)]'
```

Returns only unresolved threads directly.

**Response field mapping (critical -- using wrong ID causes silent failures):**

| Field              | Format                  | Use for                      |
|--------------------|-------------------------|------------------------------|
| thread `.id`       | `PRRT_...` (node ID)    | `resolveReviewThread` mutation |
| comment `.databaseId` | `2949637341` (numeric)  | REST reply endpoint          |
| comment `.id`      | `PRRC_...` (node ID)    | Not typically needed          |

**GitLab (REST):**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

### Reply and Resolve (One Batched Command)

**GitHub** -- combine all REST replies and a batch GraphQL resolve mutation into one `&&`-chained command. Reply uses `databaseId` (numeric), resolve uses thread `id` (PRRT_ node ID):

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{databaseId_1}/replies -f body="Fixed in {sha} -- {explanation}" && \
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{databaseId_2}/replies -f body="Addressed in {sha}" && \
gh api graphql -f query="
mutation {
  t1: resolveReviewThread(input: {threadId: \"PRRT_thread1\"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: \"PRRT_thread2\"}) { thread { isResolved } }
}"
```

**GitLab:**
```bash
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1}/notes --method POST --field "body=Fixed in {sha}" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1} --method PUT --field "resolved=true" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2}/notes --method POST --field "body=Addressed" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2} --method PUT --field "resolved=true"
```

---

## Line-Specific Comments (Write)

> **Reference**: See `references/line-comments.md` for full patterns: single-line, multi-line range, replies, edit/delete, batch reviews.

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="{comment}" -f path="{file}" -F line={line} -f side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json headRefOid --jq .headRefOid)"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --method POST \
  --field "body={comment}" \
  --field "position[base_sha]={base_sha}" \
  --field "position[head_sha]={head_sha}" \
  --field "position[start_sha]={base_sha}" \
  --field "position[position_type]=text" \
  --field "position[new_path]={file}" \
  --field "position[new_line]={line}"
```

---

## Read-Only API Endpoints

> **Reference**: See `references/api-readonly.md` for the full list of read-only REST and GraphQL endpoints (PR data, issue timelines, review threads) for both providers.

---

## Key `glab` vs `gh` Differences

| Aspect                | `gh`                                    | `glab`                                      |
|-----------------------|-----------------------------------------|---------------------------------------------|
| MR description flag   | `--body`                                | `--description`                             |
| Delete source branch  | `--delete-branch`                       | `--remove-source-branch`                    |
| JSON output           | `--json field1,field2` (filtered)       | `-F json` (full resource)                   |
| jq filtering          | `--jq '.expr'` (native)                | pipe to `\| jq '.expr'`                    |
| Squash on create      | `--squash`                              | `--squash-before-merge`                     |
| GraphQL               | `gh api graphql -f query='...'`         | Not supported (REST only)                   |
| Thread resolution     | GraphQL mutation                        | `PUT /discussions/:id` with `resolved=true` |
| Approval model        | Review states (APPROVED, CHANGES_REQUESTED) | Approval rules + approve/revoke        |
| Pagination            | `--paginate` (subcommands + api)        | `--paginate` (api only), `-P` (subcommands) |

---

## Allowlist

> **Reference**: See `references/allowlist.md` for tiered `Bash(command:*)` patterns covering all read-only operations -- safe to auto-approve in Claude Code `settings.json` or OpenCode config.
