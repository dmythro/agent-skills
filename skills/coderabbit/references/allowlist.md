# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json` covering CodeRabbit CLI commands.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches the command prefix with any arguments (including none)
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`)
- Patterns cover both the full binary name and the `cr` alias where included

## Recommended: Broad Patterns

Read-only or local-only commands -- no code leaves the machine, no quota is consumed:

```json
{
  "permissions": {
    "allow": [
      "Bash(coderabbit auth status)",
      "Bash(coderabbit doctor)",
      "Bash(coderabbit stats:*)",
      "Bash(coderabbit review findings)",
      "Bash(cr auth status)",
      "Bash(cr doctor)",
      "Bash(cr stats:*)",
      "Bash(cr review findings)"
    ]
  }
}
```

### Why These Are Safe

- `auth status` -- exact match (no `:*`), reports login state only
- `doctor` -- local diagnostics (runtime, storage, git metadata, connectivity)
- `stats` -- local review statistics
- `review findings` -- replays the **cached** findings from the last review; no new analysis, no upload, no quota

## Review Runs (Opt-In)

Running a review uploads the diff to CodeRabbit's service and consumes an hourly-quota slot. Allowlist these only after confirming both are acceptable for the repos this config applies to (project-level `.claude/settings.json` is the right scope, not global):

```json
{
  "permissions": {
    "allow": [
      "Bash(coderabbit review --type:*)",
      "Bash(cr review --type:*)"
    ]
  }
}
```

Scoping to the `--type` prefix keeps the canonical review invocations (`coderabbit review --type uncommitted --agent`, etc.) auto-approved while a bare `coderabbit review` (unscoped default) still prompts -- a nudge toward deliberate scope selection.

## Not Included (Manual Approval Required)

- **`coderabbit auth login` / `logout` / `auth org`** -- credential and org-context changes
- **`coderabbit update`** -- self-modifying binary update
- **`coderabbit review --api-key ...`** -- inline credentials should never be auto-approved
- **Bare `coderabbit` / `cr`** -- runs an unscoped review of all changes; make scope explicit instead
