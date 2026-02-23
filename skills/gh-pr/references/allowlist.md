# Suggested Allowlist Patterns

Copy-paste ready `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config. Covers all read-only `gh` operations from this skill.

## Pull Requests

```json
"Bash(gh pr view:*)",
"Bash(gh pr list:*)",
"Bash(gh pr checks:*)",
"Bash(gh pr diff:*)",
"Bash(gh pr status:*)"
```

## Issues

```json
"Bash(gh issue view:*)",
"Bash(gh issue list:*)",
"Bash(gh issue status:*)"
```

## Actions & Runs

```json
"Bash(gh run view:*)",
"Bash(gh run list:*)"
```

## Search

```json
"Bash(gh search prs:*)",
"Bash(gh search issues:*)"
```

## Repo

```json
"Bash(gh repo view:*)"
```

## All Patterns Combined

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
      "Bash(gh run view:*)",
      "Bash(gh run list:*)",
      "Bash(gh search prs:*)",
      "Bash(gh search issues:*)",
      "Bash(gh repo view:*)"
    ]
  }
}
```

## Note on `gh api`

`gh api` calls are **not included** in the allowlist because they can be both read and write operations. The `Bash(command:*)` pattern can't distinguish `gh api GET` from `gh api --method POST`. Let these go through the normal approval flow.
