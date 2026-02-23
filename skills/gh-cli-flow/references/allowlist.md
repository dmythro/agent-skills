# Suggested Allowlist Patterns

Copy-paste ready `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config. Covers all read-only `gh` operations.

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

## Workflows

```json
"Bash(gh workflow list:*)",
"Bash(gh workflow view:*)"
```

## Search

```json
"Bash(gh search prs:*)",
"Bash(gh search issues:*)",
"Bash(gh search code:*)",
"Bash(gh search commits:*)",
"Bash(gh search repos:*)"
```

## Labels

```json
"Bash(gh label list:*)"
```

## Repo

```json
"Bash(gh repo view:*)",
"Bash(gh repo list:*)"
```

## Releases

```json
"Bash(gh release view:*)",
"Bash(gh release list:*)"
```

## CI Config

```json
"Bash(gh variable list:*)",
"Bash(gh variable get:*)",
"Bash(gh secret list:*)",
"Bash(gh cache list:*)"
```

## Rulesets

```json
"Bash(gh ruleset list:*)",
"Bash(gh ruleset view:*)",
"Bash(gh ruleset check:*)"
```

## Auth & Config

```json
"Bash(gh auth status:*)",
"Bash(gh config list:*)",
"Bash(gh config get:*)"
```

## Projects

```json
"Bash(gh project list:*)",
"Bash(gh project view:*)"
```

## Misc

```json
"Bash(gh gist list:*)",
"Bash(gh gist view:*)",
"Bash(gh extension list:*)",
"Bash(gh org list:*)",
"Bash(gh ssh-key list:*)",
"Bash(gh gpg-key list:*)",
"Bash(gh codespace list:*)"
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
      "Bash(gh workflow list:*)",
      "Bash(gh workflow view:*)",
      "Bash(gh search prs:*)",
      "Bash(gh search issues:*)",
      "Bash(gh search code:*)",
      "Bash(gh search commits:*)",
      "Bash(gh search repos:*)",
      "Bash(gh label list:*)",
      "Bash(gh repo view:*)",
      "Bash(gh repo list:*)",
      "Bash(gh release view:*)",
      "Bash(gh release list:*)",
      "Bash(gh variable list:*)",
      "Bash(gh variable get:*)",
      "Bash(gh secret list:*)",
      "Bash(gh cache list:*)",
      "Bash(gh ruleset list:*)",
      "Bash(gh ruleset view:*)",
      "Bash(gh ruleset check:*)",
      "Bash(gh auth status:*)",
      "Bash(gh config list:*)",
      "Bash(gh config get:*)",
      "Bash(gh project list:*)",
      "Bash(gh project view:*)",
      "Bash(gh gist list:*)",
      "Bash(gh gist view:*)",
      "Bash(gh extension list:*)",
      "Bash(gh org list:*)",
      "Bash(gh ssh-key list:*)",
      "Bash(gh gpg-key list:*)",
      "Bash(gh codespace list:*)"
    ]
  }
}
```

## Not Included

`gh api` calls are **not included** because they can be both read and write operations. The `Bash(command:*)` pattern cannot distinguish `gh api GET` from `gh api --method POST`. Let these go through the normal approval flow.

Write operations (`gh pr create`, `gh pr merge`, `gh issue create`, `gh run rerun`, `gh workflow run`, `gh label create`, etc.) are also excluded -- these modify state and should require explicit approval.
