# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers read-only CI/CD operations for GitHub Actions (`gh`) and GitLab CI (`glab`).

**OpenCode**: Same commands work with picomatch format (`"command": "allow"`) in OpenCode config. See the previous version of this file for full OpenCode examples.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches command prefix with any arguments (including none). This is the current recommended syntax for both `gh` and `glab` commands.
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`)

---

## Recommended: Broad Patterns

Match any read-only CI subcommand variation regardless of `--json` fields or flags. These subcommands are inherently read-only.

### Claude Code

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr checks:*)",
      "Bash(gh run view:*)",
      "Bash(gh run list:*)",
      "Bash(gh workflow list:*)",
      "Bash(gh workflow view:*)",
      "Bash(gh variable list:*)",
      "Bash(gh secret list:*)",
      "Bash(gh cache list:*)",
      "Bash(gh ruleset list:*)",
      "Bash(gh ruleset view:*)",
      "Bash(gh ruleset check:*)",
      "Bash(gh auth status)",
      "Bash(glab ci status:*)",
      "Bash(glab ci get:*)",
      "Bash(glab ci list:*)",
      "Bash(glab ci view:*)",
      "Bash(glab ci trace:*)",
      "Bash(glab auth status)"
    ]
  }
}
```

### Why Broad Patterns Are Safe

- `gh pr checks`, `gh run view`, `gh run list` -- **read-only subcommands**, no flag combination can cause writes
- `gh workflow list`, `gh workflow view` -- list/view only, not `gh workflow run` (which triggers execution)
- `gh variable list`, `gh secret list`, `gh cache list` -- read-only listing (names only, not values)
- `gh ruleset list/view/check` -- read-only inspection
- `glab ci status/get/list/view/trace` -- all read-only CI queries
- `gh auth status`, `glab auth status` -- exact match (no `:*`) because `--show-token` flag would expose credentials
- Shell operator awareness prevents injection

---

## Not Included (Manual Approval Required)

- **`gh run rerun`** -- re-runs workflow, consumes CI minutes
- **`gh run cancel`** -- cancels running workflow
- **`gh workflow run`** -- triggers workflow dispatch
- **`gh run delete`** -- deletes workflow run logs
- **`glab ci retry`** -- retries failed pipeline
- **`glab ci cancel`** -- cancels running pipeline
- **`gh variable get`** -- retrieves variable values, which may contain secrets
- **`glab variable list`** -- exposes CI/CD variable values, which often contain secrets (API keys, passwords)
- **All `gh api` / `glab api` calls** -- cannot distinguish read from write by pattern alone

---

## Alternative: Strict Exact Patterns

For maximum restriction, use exact command strings that only auto-approve specific field sets.

### Tier 1: Current-Branch (No Wildcards)

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
"Bash(glab auth status)"
```

### Tier 2: By-ID (Single `*` for Run/Pipeline ID)

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
