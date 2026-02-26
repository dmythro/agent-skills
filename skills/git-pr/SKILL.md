---
name: git-pr
description: PR and MR workflows for GitHub (gh) and GitLab (glab). Covers creation,
  review comment handling, line-specific comments, review state queries, and merging
---

# PR and MR Workflows

**Primary skill for pull request and merge request workflows across GitHub and GitLab.** Covers the full lifecycle: creation, review queries, comment handling, line-specific comments, and merging. All recipes use minimal field sets for token efficiency.

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
2. **Use `--json field1,field2` with `gh` to filter output.** This IS the efficiency mechanism -- no `--jq` needed for basic queries.
3. **`glab` has no `--json field1,field2` equivalent.** Use `-F json | jq '{fields}'` to filter output for token efficiency.

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

> **Reference**: See `references/pr-comment-workflow.md` for the full opinionated workflow -- fetch unresolved threads, evaluate each comment critically, fix/reply/resolve.

Quick summary:
1. Fetch unresolved review threads (Tier 3 -- manual approval)
2. For each thread: read the code context, check if already addressed
3. Decide: fix code + reply + resolve, or reply with evidence + resolve, or leave for discussion
4. Don't blindly agree -- validate comments against actual code and conventions

### Fetch Unresolved Threads

**GitHub (GraphQL):**
```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          id, isResolved, isOutdated, path, line, startLine
          comments(first: 10) {
            nodes { id, databaseId, body, author { login } }
          }
        }
      }
    }
  }
}' -f owner={owner} -f repo={repo} -F pr={number}
```

**GitLab (REST):**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

### Reply and Resolve

**GitHub:**
```bash
# Reply to thread
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Fixed -- {explanation}."

# Resolve thread
gh api graphql -f query='
mutation($id: ID!) {
  resolveReviewThread(input: {threadId: $id}) { thread { isResolved } }
}' -f id="{thread_node_id}"
```

**GitLab:**
```bash
# Reply to discussion
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Fixed -- {explanation}."

# Resolve discussion
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
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
