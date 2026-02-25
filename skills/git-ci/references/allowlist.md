# Suggested Allowlist Patterns

Tiered auto-approval patterns for Claude Code `settings.json` and OpenCode config. Covers read-only CI/CD operations for GitHub Actions (`gh`) and GitLab CI (`glab`).

## Pattern Syntax

- **Claude Code**: `Bash(exact command)` or `Bash(cmd *)` -- glob matching, shell-operator aware (`*` cannot match `&&`, `||`, `;`, `|`)
- **OpenCode**: `"exact command": "allow"` -- uses picomatch(), last-match-wins
- **Deprecated**: `:*` syntax. Use space-star (` *`) instead.

---

## Tier 1: Exact (No Wildcards)

Current-branch commands. Auto-approve with zero risk.

### Claude Code

```json
"Bash(gh pr checks --json name,state,conclusion,bucket)",
"Bash(gh pr checks --watch --fail-fast)",
"Bash(gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit 10)",
"Bash(gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus)",
"Bash(gh workflow list --json id,name,state)",
"Bash(gh variable list)",
"Bash(gh secret list)",
"Bash(gh cache list)",
"Bash(gh ruleset list)",
"Bash(gh auth status)",
"Bash(glab ci status)",
"Bash(glab ci get)",
"Bash(glab ci list)",
"Bash(glab ci status --live)",
"Bash(glab variable list)",
"Bash(glab auth status)"
```

### OpenCode

```json
"gh pr checks --json name,state,conclusion,bucket": "allow",
"gh pr checks --watch --fail-fast": "allow",
"gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit 10": "allow",
"gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus": "allow",
"gh workflow list --json id,name,state": "allow",
"gh variable list": "allow",
"gh secret list": "allow",
"gh cache list": "allow",
"gh ruleset list": "allow",
"gh auth status": "allow",
"glab ci status": "allow",
"glab ci get": "allow",
"glab ci list": "allow",
"glab ci status --live": "allow",
"glab variable list": "allow",
"glab auth status": "allow"
```

---

## Tier 2: Parametric (Single `*` for Run/Pipeline ID)

Slightly broader but still safe. Add these to Tier 1 patterns.

### Claude Code

```json
"Bash(gh pr checks * --json name,state,conclusion,bucket)",
"Bash(gh pr checks * --watch --fail-fast)",
"Bash(gh run view * --json jobs,status,conclusion,displayTitle)",
"Bash(gh run view * --log-failed)",
"Bash(gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit *)",
"Bash(gh ruleset view *)",
"Bash(gh ruleset check *)",
"Bash(glab ci view *)",
"Bash(glab ci trace *)",
"Bash(glab mr view -F json | jq *)",
"Bash(glab mr view * -F json | jq *)"
```

---

## All Patterns Combined

### Claude Code

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr checks --json name,state,conclusion,bucket)",
      "Bash(gh pr checks --watch --fail-fast)",
      "Bash(gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit 10)",
      "Bash(gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus)",
      "Bash(gh workflow list --json id,name,state)",
      "Bash(gh variable list)",
      "Bash(gh secret list)",
      "Bash(gh cache list)",
      "Bash(gh ruleset list)",
      "Bash(gh auth status)",
      "Bash(glab ci status)",
      "Bash(glab ci get)",
      "Bash(glab ci list)",
      "Bash(glab ci status --live)",
      "Bash(glab variable list)",
      "Bash(glab auth status)",
      "Bash(gh pr checks * --json name,state,conclusion,bucket)",
      "Bash(gh pr checks * --watch --fail-fast)",
      "Bash(gh run view * --json jobs,status,conclusion,displayTitle)",
      "Bash(gh run view * --log-failed)",
      "Bash(gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit *)",
      "Bash(gh ruleset view *)",
      "Bash(gh ruleset check *)",
      "Bash(glab ci view *)",
      "Bash(glab ci trace *)",
      "Bash(glab mr view -F json | jq *)",
      "Bash(glab mr view * -F json | jq *)"
    ]
  }
}
```

### OpenCode

```json
{
  "gh pr checks --json name,state,conclusion,bucket": "allow",
  "gh pr checks --watch --fail-fast": "allow",
  "gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit 10": "allow",
  "gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus": "allow",
  "gh workflow list --json id,name,state": "allow",
  "gh variable list": "allow",
  "gh secret list": "allow",
  "gh cache list": "allow",
  "gh ruleset list": "allow",
  "gh auth status": "allow",
  "glab ci status": "allow",
  "glab ci get": "allow",
  "glab ci list": "allow",
  "glab ci status --live": "allow",
  "glab variable list": "allow",
  "glab auth status": "allow"
}
```

---

## Not Included (Tier 3 -- Manual Approval)

These require explicit user approval:

- **`gh run rerun`** -- re-runs workflow, consumes CI minutes
- **`gh run cancel`** -- cancels running workflow
- **`gh workflow run`** -- triggers workflow dispatch
- **`gh run delete`** -- deletes workflow run logs
- **`glab ci retry`** -- retries failed pipeline
- **`glab ci cancel`** -- cancels running pipeline
- **All `gh api` / `glab api` calls** -- cannot distinguish read from write by pattern
