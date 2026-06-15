# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers read-only `gh` and `glab` PR/MR operations.

**OpenCode**: Same commands work with picomatch format (`"command": "allow"`) in OpenCode config. See the previous version of this file for full OpenCode examples.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches command prefix with any arguments (including none). This is the current recommended syntax for both `gh` and `glab` commands.
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`) -- pipe-inclusive patterns must spell out the pipe explicitly
- For `glab` commands piped to `jq`, use `Bash(glab ... | jq *)` because `*` cannot cross the pipe boundary
- **Variable assignments break matching** -- any `VAR=value` line before the command (inline or separate line) makes the command string start with `VAR=...`. Use `$(...)` command substitution directly in the command arguments instead
- **Single-line commands recommended** -- `*` may not match across newlines (undocumented). Generate commands that need allowlist matching as a single line to be safe

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

```text
"Bash(gh api graphql -f query=*{ repository*)",
"Bash(gh api repos/*/pulls/*/comments)",
"Bash(gh api repos/*/pulls/*/comments --paginate)",
"Bash(gh api repos/*/pulls/*/comments --jq *)",
"Bash(gh api repos/*/pulls/*/comments --paginate --jq *)",
"Bash(gh api repos/*/pulls/*/reviews)",
"Bash(gh api repos/*/pulls/*/reviews --paginate)",
"Bash(gh api repos/*/pulls/*/reviews --jq *)",
"Bash(gh api repos/*/pulls/*/reviews --paginate --slurp | jq *)",
"Bash(gh api repos/*/pulls/*/files *)",
"Bash(gh api repos/*/pulls/*/commits *)",
"Bash(gh api repos/*/pulls/*/requested_reviewers)",
"Bash(gh api repos/*/issues/*/comments)",
"Bash(gh api repos/*/issues/*/comments --paginate)",
"Bash(gh api repos/*/issues/*/comments --jq *)",
"Bash(gh api repos/*/issues/*/comments --paginate --slurp | jq *)",
"Bash(gh api repos/*/issues/*/labels)",
"Bash(gh api repos/*/issues/*/labels --jq *)",
"Bash(gh api repos/*/issues/*/timeline *)"
```

**Pattern details:**

- **GraphQL `*{ repository*`**: matches single-line read queries containing `{ repository`. The `{` prefix prevents matching `repositoryId` or similar strings in mutations. The command should be a single line -- `*` may not match across newlines reliably. Use `$(...)` substitution for owner/repo inline
- **`/files` and `/commits`**: trailing `*` allows any flags -- safe because these are GET-only endpoints
- **`/comments`, `/reviews`, `/requested_reviewers`**: bare pattern (no trailing `*`) blocks POST/DELETE. Explicit `--paginate`, `--jq *`, and `--paginate --slurp | jq *` variants are added separately for read-only flag support
- **`--paginate --slurp | jq *`** (reviews, issues comments): the Copilot loop slurps all pages into one array and pipes to a separate `jq`, because `--paginate` applies `-q/--jq` per page (and `--slurp` cannot combine with `-q/--jq`). The pipe is spelled out because `*` cannot cross it
- **Why not trailing `*` on `/comments`**: `gh api repos/.../comments -f body="text"` would match -- that's a POST. Enumerating safe flags (`--paginate`, `--jq`, `--slurp | jq`) is safer

---

## Bot Re-Request (Opt-In Write)

The bot review loop (`references/bot-review-loop.md`) re-requests a review each round. These are the **only writes** this skill suggests auto-approving -- a Copilot reviewer request, or a CodeRabbit `@coderabbitai review` comment. Neither merges or closes the PR, but both patterns key on the command *ending* (a `*` wildcard precedes it; see Caveat), so review them before trusting. The read-only polling (`gh api .../reviews`, `gh pr view --json headRefOid`, the GraphQL `reviewThreads` query) is already covered by the patterns above.

### Claude Code

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr edit * --add-reviewer \"@copilot\")",
      "Bash(gh pr comment * --body \"@coderabbitai review\")"
    ]
  }
}
```

- **Issue each in its canonical form**: Copilot `gh pr edit {N} --add-reviewer "@copilot"` (quoted, flag last); CodeRabbit `gh pr comment {N} --body "@coderabbitai review"`. The loop depends on these exact forms; reordering or dropping quotes will prompt.
- **Scope**: each matches only commands **ending** in the shown flag/body. A command where it isn't last (e.g. `--add-reviewer "@copilot" --title X`) is not matched and stays manual. The CodeRabbit pattern is scoped to the exact trigger text, so it can't auto-approve arbitrary `gh pr comment` bodies.
- **Caveat**: because matching keys on the *ending*, extra flags placed before it (e.g. `--title`, `--body` on `gh pr edit`) would also be auto-approved and **could modify PR content**. The loop only issues the bare canonical forms; for zero latitude, omit these and approve manually.
- Removal/cleanup commands (`gh pr edit --remove-reviewer "@copilot"`) are not allowlisted -- not part of the loop.

---

## Not Included (Manual Approval Required)

- **GraphQL mutations** -- `resolveReviewThread`, `addPullRequestReviewComment`, etc. use `mutation {` which does not match the `*{ repository*` allowlist pattern
- **REST writes** -- POST/PUT/DELETE on `/comments`, `/reviews`, `/requested_reviewers`
- **Write subcommands** -- `gh pr create`, `gh pr merge`, `gh pr review`, `glab mr create`, `glab mr merge`, `glab mr approve`
- **PR edits** -- `gh pr edit` (title, body, base, reviewers) stays manual, except the narrowly-scoped Copilot re-request documented above under "Copilot Re-Request (Opt-In Write)"
- **Comment operations** -- replies, line comments, review submissions
- **Thread resolution** -- GraphQL mutations, `glab api --method PUT`

**Batching for speed**: All replies and resolves should be combined into a single `&&`-chained command so the user approves once. See `references/pr-comment-workflow.md` for the batched pattern.

---

## Alternative: Strict Exact Patterns

For maximum restriction, use exact command strings. These only auto-approve the specific `--json` field sets listed below and prompt for any variation. Useful when you want to control exactly which data the agent can query.

### Tier 1: Current-Branch (No Wildcards)

```text
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

```text
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
