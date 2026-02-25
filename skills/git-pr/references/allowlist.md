# Suggested Allowlist Patterns

Tiered auto-approval patterns for Claude Code `settings.json` and OpenCode config. Covers read-only `gh` and `glab` PR/MR operations.

## Pattern Syntax

- **Claude Code**: `Bash(exact command)` or `Bash(cmd *)` -- glob matching, shell-operator aware (`*` cannot match `&&`, `||`, `;`, `|`)
- **OpenCode**: `"exact command": "allow"` -- uses picomatch(), last-match-wins
- **Deprecated**: `:*` syntax. Use space-star (` *`) instead.

---

## Tier 1: Exact (No Wildcards)

Current-branch commands. Auto-approve with zero risk.

### Claude Code

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
"Bash(gh auth status)",
"Bash(glab mr view -F json | jq *)",
"Bash(glab mr diff)",
"Bash(glab mr list -F json | jq *)",
"Bash(glab issue list -F json | jq *)",
"Bash(glab auth status)"
```

### OpenCode

```json
"gh pr view --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName": "allow",
"gh pr view --json reviews,reviewRequests,latestReviews": "allow",
"gh pr view --json files": "allow",
"gh pr view --json commits": "allow",
"gh pr view --json labels,milestone": "allow",
"gh pr view --json closingIssuesReferences": "allow",
"gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus": "allow",
"gh pr diff --name-only": "allow",
"gh pr list --json number,title,author,reviewDecision,updatedAt": "allow",
"gh pr status": "allow",
"gh issue list --json number,title,state,author,labels": "allow",
"gh label list": "allow",
"gh repo view --json name,owner,defaultBranchRef": "allow",
"gh auth status": "allow",
"glab mr view -F json | jq *": "allow",
"glab mr diff": "allow",
"glab mr list -F json | jq *": "allow",
"glab issue list -F json | jq *": "allow",
"glab auth status": "allow"
```

### Notes on glab Patterns

`glab` has no `--json field1,field2` equivalent. Patterns use `| jq *` because Claude Code is shell-operator aware -- the pipe is literal and `*` only matches the jq expression. If using plain `glab mr view -F json` (full output), replace `| jq *` entries with the command without pipe.

---

## Tier 2: Parametric (Single `*` for PR/MR Number)

Slightly broader but still safe. Add these to Tier 1 patterns.

### Claude Code

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
"Bash(gh search code *)",
"Bash(glab mr view * -F json | jq *)",
"Bash(glab issue view * -F json | jq *)"
```

---

## All Patterns Combined

### Claude Code

```json
{
  "permissions": {
    "allow": [
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
      "Bash(gh auth status)",
      "Bash(glab mr view -F json | jq *)",
      "Bash(glab mr diff)",
      "Bash(glab mr list -F json | jq *)",
      "Bash(glab issue list -F json | jq *)",
      "Bash(glab auth status)",
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
      "Bash(gh search code *)",
      "Bash(glab mr view * -F json | jq *)",
      "Bash(glab issue view * -F json | jq *)"
    ]
  }
}
```

### OpenCode

```json
{
  "gh pr view --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName": "allow",
  "gh pr view --json reviews,reviewRequests,latestReviews": "allow",
  "gh pr view --json files": "allow",
  "gh pr view --json commits": "allow",
  "gh pr view --json labels,milestone": "allow",
  "gh pr view --json closingIssuesReferences": "allow",
  "gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus": "allow",
  "gh pr diff --name-only": "allow",
  "gh pr list --json number,title,author,reviewDecision,updatedAt": "allow",
  "gh pr status": "allow",
  "gh issue list --json number,title,state,author,labels": "allow",
  "gh label list": "allow",
  "gh repo view --json name,owner,defaultBranchRef": "allow",
  "gh auth status": "allow",
  "glab mr view -F json | jq *": "allow",
  "glab mr diff": "allow",
  "glab mr list -F json | jq *": "allow",
  "glab issue list -F json | jq *": "allow",
  "glab auth status": "allow"
}
```

---

## Not Included (Tier 3 -- Manual Approval)

These require explicit user approval:

- **All `gh api graphql` calls** -- GraphQL always uses POST, cannot distinguish read from write
- **All `gh api repos/*/pulls/*/comments` calls** -- can be GET or POST depending on flags
- **Write operations**: `gh pr create`, `gh pr merge`, `gh pr review`, `glab mr create`, `glab mr merge`, `glab mr approve`
- **Thread resolution**: `gh api graphql mutation`, `glab api --method PUT`
- **Comment operations**: replies, line comments, review submissions
