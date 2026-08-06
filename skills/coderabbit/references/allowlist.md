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
      "Bash(coderabbit review findings:*)",
      "Bash(coderabbit config validate:*)",
      "Bash(cr auth status)",
      "Bash(cr doctor)",
      "Bash(cr stats:*)",
      "Bash(cr review findings:*)",
      "Bash(cr config validate:*)"
    ]
  }
}
```

### Why These Are Safe

- `auth status` -- exact match (no `:*`), reports login state only
- `doctor` -- local diagnostics (runtime, storage, git metadata, connectivity)
- `stats` -- local review statistics
- `review findings` -- replays the **cached** findings from the last review (`:*` covers `--dir <path>`); no new analysis, no upload, no quota
- `config validate` -- checks a local `.coderabbit.yaml` against the official schema; no review, no code upload

## Review Runs (Opt-In)

Running a review uploads the diff to CodeRabbit's service and consumes an hourly-quota slot. Allowlist these only after confirming both are acceptable for the repos this config applies to (project-level `.claude/settings.json` is the right scope, not global):

```json
{
  "permissions": {
    "allow": [
      "Bash(coderabbit review --committed:*)",
      "Bash(coderabbit review --uncommitted:*)",
      "Bash(coderabbit review --base:*)",
      "Bash(cr review --committed:*)",
      "Bash(cr review --uncommitted:*)",
      "Bash(cr review --base:*)"
    ]
  }
}
```

Keying on an explicit scope flag in first position (`--committed`, `--uncommitted`, or `--base` for full-branch reviews) keeps the canonical invocations (`coderabbit review --committed --base main --agent`, etc.) auto-approved while a bare `coderabbit review` still prompts -- the same deliberate-scope nudge the pre-0.7 `--type` prefix patterns provided.

**Caveat**: `:*` matches all remaining arguments, so a review command that appends `--api-key <key>` would also auto-approve -- prefix patterns cannot exclude a later flag (the pre-0.7 `--type` patterns had the same limitation; compare the re-request Caveat in the `git-pr` allowlist). Stored auth (`coderabbit auth login`) makes inline keys unnecessary on a developer machine; in environments where inline keys actually appear (headless/CI), omit these patterns and approve review runs manually.

## Not Included (Manual Approval Required)

- **`coderabbit auth login` / `logout` / `auth org`** -- credential and org-context changes
- **`coderabbit update`** -- self-modifying binary update
- **`coderabbit skills`** -- installs/updates CodeRabbit's agent skills (writes outside the repo)
- **`coderabbit review --api-key ...`** -- inline credentials: never add a pattern targeting them, and note the opt-in review prefixes above cannot *exclude* them (see their Caveat); skip those patterns where inline keys are a real risk
- **Bare `coderabbit` / `cr`** -- runs an unscoped review of all changes; make scope explicit instead
