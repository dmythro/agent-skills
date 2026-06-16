# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers read-only `gh project` and project-read GraphQL. Project **writes** are intentionally left out (they create/modify boards and issues).

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches the command prefix with any arguments.
- `*` cannot cross shell operators (`&&`, `|`, `;`) or, reliably, newlines -- keep allowlisted commands single-line.
- Avoid `VAR=...` prefixes (they break matching); use `$(...)` inline, or gh's `{owner}/{repo}` placeholders on REST endpoints.

## Recommended: Broad Patterns (Read-Only)

```json
{
  "permissions": {
    "allow": [
      "Bash(gh project list:*)",
      "Bash(gh project view:*)",
      "Bash(gh project field-list:*)",
      "Bash(gh project item-list:*)",
      "Bash(gh issue list:*)",
      "Bash(gh issue view:*)",
      "Bash(gh label list:*)",
      "Bash(gh api repos/*/issues/*/sub_issues)",
      "Bash(gh api repos/*/issues/*/sub_issues --jq *)",
      "Bash(gh api repos/*/issues/*/sub_issues --paginate --jq *)",
      "Bash(gh api graphql -f query=*{ viewer { projectV2*)",
      "Bash(gh api graphql -f query=*{ organization(login*)"
    ]
  }
}
```

### Why these are safe

- `gh project list/view/field-list/item-list` -- query-only subcommands; no flag turns them into writes. These cover ID discovery (project id, field ids, option ids, item ids).
- `gh issue list/view`, `gh label list` -- read-only (shared with the `git-pr` skill).
- `repos/*/issues/*/sub_issues` (bare / `--jq` / `--paginate --jq`) -- the GET reads a parent's children; POST/DELETE are excluded by enumerating only read flags.
- `*{ viewer { projectV2*` -- matches single-line project **read** queries (views, workflows, fields). Mutations begin with `mutation` and don't match. The included `*{ organization(login*` is the org-owned variant: the query is `organization(login: "ORG") { projectV2 }`, so a `{ organization { projectV2` pattern would *not* match (the `(login: ...)` argument sits in between).

## Not Included (Manual Approval Required)

Every project write -- they change boards, fields, issues, or links:

- **Project/field/item writes** -- `gh project create`, `copy`, `edit`, `link`, `field-create`, `item-add`, `item-edit`, `item-archive`, `item-delete`
- **Issue/label writes** -- `gh issue create`/`edit`, `gh label create`
- **Sub-issue links** -- `POST`/`DELETE` on `.../sub_issues` (`--method POST|DELETE`)
- **GraphQL mutations** -- `updateProjectV2ItemFieldValue`, `updateProjectV2ItemPosition`, `addSubIssue`, etc. (`mutation {`)

### Opt-in: the status-flow edit

Driving the board (`gh project item-edit` to set Status/Priority) is the one write the loop repeats. It's a write and stays manual by default. To run the board hands-off, you can opt into:

```json
"Bash(gh project item-edit:*)"
```

Caveat: this is broad -- `item-edit` can set **any** field on **any** item in your projects, not just Status/Priority. Add it only if you accept that latitude; otherwise approve each edit.
