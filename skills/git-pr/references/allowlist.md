# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers read-only `gh` and `glab` PR/MR operations.

**OpenCode**: Same commands work with picomatch format (`"command": "allow"`) in OpenCode config. See the previous version of this file for full OpenCode examples.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches command prefix with any arguments (including none). This is the current recommended syntax for both `gh` and `glab` commands.
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`) -- pipe-inclusive patterns must spell out the pipe explicitly
- For `glab` commands piped to `jq`, use `Bash(glab ... | jq *)` because `*` cannot cross the pipe boundary

---

## Recommended: Broad Patterns

Match any read-only subcommand variation regardless of `--json` fields or flags. These subcommands are inherently read-only -- no flag combination can cause writes.

### Claude Code

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr view:*)",
      "Bash(gh pr list:*)",
      "Bash(gh pr checks:*)",
      "Bash(gh pr diff:*)",
      "Bash(gh pr status:*)",
      "Bash(gh issue view:*)",
      "Bash(gh issue list:*)",
      "Bash(gh issue status:*)",
      "Bash(gh search prs:*)",
      "Bash(gh search issues:*)",
      "Bash(gh search code:*)",
      "Bash(gh search commits:*)",
      "Bash(gh label list:*)",
      "Bash(gh repo view:*)",
      "Bash(gh repo list:*)",
      "Bash(gh auth status)",
      "Bash(gh config list:*)",
      "Bash(gh config get:*)",
      "Bash(git remote get-url origin)",
      "Bash(glab mr view:*)",
      "Bash(glab mr diff:*)",
      "Bash(glab mr list:*)",
      "Bash(glab issue view:*)",
      "Bash(glab issue list:*)",
      "Bash(glab auth status)",
      "Bash(glab mr view -F json | jq *)",
      "Bash(glab mr view * -F json | jq *)",
      "Bash(glab mr list -F json | jq *)",
      "Bash(glab issue list -F json | jq *)"
    ]
  }
}
```

### Why Broad Patterns Are Safe

- `gh pr view`, `gh pr list`, `gh pr checks`, `gh pr diff`, `gh pr status` -- **read-only subcommands**, no `--json` field combination can cause writes
- `gh issue view`, `gh issue list`, `gh issue status` -- read-only
- `gh search *`, `gh label list`, `gh repo view` -- query-only
- `gh auth status`, `glab auth status` -- exact match (no `:*`) because `--show-token` flag would expose credentials
- `glab mr view`, `glab mr list`, `glab mr diff` -- read-only
- Shell operator awareness prevents injection (e.g., `gh pr view && rm -rf /` cannot match)

---

## GitHub API Read-Only Patterns

`gh api` calls require specific patterns because the same command can perform reads or writes. These restrict to known GET-only endpoints.

```json
"Bash(gh api graphql -f query=*repository(*))",
"Bash(gh api repos/*/pulls/*/comments)",
"Bash(gh api repos/*/pulls/*/reviews)",
"Bash(gh api repos/*/pulls/*/files *)",
"Bash(gh api repos/*/pulls/*/commits *)",
"Bash(gh api repos/*/pulls/*/requested_reviewers)",
"Bash(gh api repos/*/issues/*/comments)",
"Bash(gh api repos/*/issues/*/labels)",
"Bash(gh api repos/*/issues/*/timeline *)"
```

**Pattern details:**

- **GraphQL `*repository(*)`**: matches read queries that access `repository(` but blocks mutations (which start with `mutation {`). The inline query format uses `{ repository(owner: ...) { ... } }` -- no GraphQL `$` variables needed
- **`/files` and `/commits`**: trailing `*` allows `--paginate` or `--jq` -- safe because these are GET-only endpoints in the GitHub API
- **`/comments`, `/reviews`, `/requested_reviewers`**: no trailing `*` to block POST/DELETE operations (which append `-f`, `--method POST`, etc.)

---

## Not Included (Manual Approval Required)

- **GraphQL mutations** -- `resolveReviewThread`, `addPullRequestReviewComment`, etc. use `mutation(` which does not match `*query(*)`
- **REST writes** -- POST/PUT/DELETE on `/comments`, `/reviews`, `/requested_reviewers`
- **Write subcommands** -- `gh pr create`, `gh pr merge`, `gh pr review`, `glab mr create`, `glab mr merge`, `glab mr approve`
- **Comment operations** -- replies, line comments, review submissions
- **Thread resolution** -- GraphQL mutations, `glab api --method PUT`

---

## Alternative: Strict Exact Patterns

For maximum restriction, use exact command strings. These only auto-approve the specific `--json` field sets listed below and prompt for any variation. Useful when you want to control exactly which data the agent can query.

### Tier 1: Current-Branch (No Wildcards)

```json
"Bash(gh pr view --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName)",
"Bash(gh pr view --json reviews,reviewRequests,latestReviews)",
"Bash(gh pr view --json files)",
"Bash(gh pr view --json commits)",
"Bash(gh pr view --json labels,milestone)",
"Bash(gh pr view --json closingIssuesReferences)",
"Bash(gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus)",
"Bash(gh pr diff --name-only)",
"Bash(gh pr list --json number,title,author,reviewDecision,updatedAt)",
"Bash(gh pr status)",
"Bash(gh issue list --json number,title,state,author,labels)",
"Bash(gh label list)",
"Bash(gh repo view --json name,owner,defaultBranchRef)",
"Bash(gh auth status)"
```

### Tier 2: By-Number (Single `*` for PR/MR Number)

```json
"Bash(gh pr view * --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName)",
"Bash(gh pr view * --json reviews,reviewRequests,latestReviews)",
"Bash(gh pr view * --json files)",
"Bash(gh pr view * --json commits)",
"Bash(gh pr view * --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus)",
"Bash(gh pr checks * --json name,state,conclusion,bucket)",
"Bash(gh pr diff * --name-only)",
"Bash(gh issue view * --json number,title,state,body,labels)",
"Bash(gh pr list --search * --json number,title,url)",
"Bash(gh search prs *)",
"Bash(gh search issues *)",
"Bash(gh search code *)"
```
