# gh project + GraphQL Reference

Validated against `gh` 2.94. Projects are addressed by **number** under an `--owner` (`@me` or an org login). `gh project` covers projects, fields, and items; **views and workflows have no create/update API** (UI-only -- see `project-setup.md`).

## Find projects and their IDs

```bash
gh project list --owner @me                        # number, title, ID (PVT_...)
gh project view <num> --owner @me                  # human summary
gh project view <num> --owner @me --format json --jq '{id, title, number}'   # the PVT_ project id
```

## Fields and their option IDs

Single-select fields (Status, Priority, Stage) are set by **option ID**, so list them first:

```bash
gh project field-list <num> --owner @me --format json \
  --jq '.fields[] | select(.type=="ProjectV2SingleSelectField") | {name, id, options: [.options[] | {name, id}]}'
```

Create a single-select field:

```bash
gh project field-create <num> --owner @me --name "Priority" \
  --data-type SINGLE_SELECT --single-select-options "P0,P1,P2,P3,P4"
```

`--data-type` is one of `TEXT | SINGLE_SELECT | DATE | NUMBER`. (A project ships with a built-in `Status` field already populated `Todo/In Progress/Done`.)

## Items (issues on the board)

```bash
gh project item-add <num> --owner @me --url <issue-url>            # add an issue/PR
gh project item-list <num> --owner @me --format json \
  --jq '.items[] | {id, title, content: .content.number}'         # item ids (PVTI_...)
```

`item-edit` needs the **item id** (`PVTI_...`), the **project id** (`PVT_...`), the **field id**, and -- for single-selects -- the **option id**:

```bash
# set a single-select (Status -> In Progress, Priority -> P1, ...)
gh project item-edit --id <itemId> --project-id <projectId> \
  --field-id <fieldId> --single-select-option-id <optionId>

# other field types
gh project item-edit --id <itemId> --project-id <projectId> --field-id <fieldId> --text "..."
gh project item-edit --id <itemId> --project-id <projectId> --field-id <fieldId> --number 3
gh project item-edit --id <itemId> --project-id <projectId> --field-id <fieldId> --date 2026-07-01
gh project item-edit --id <itemId> --project-id <projectId> --field-id <fieldId> --clear     # unset
```

### Resolve an issue's item id (one-liner)

```bash
gh project item-list <num> --owner @me --format json \
  --jq --arg n <issueNumber> '.items[] | select(.content.number==($n|tonumber)) | .id'
```

## Roadmap ordering

There is no `gh` subcommand to reorder items; use the GraphQL mutation (move an item just after another, or to the top with `afterId` omitted):

```bash
gh api graphql -f query='mutation($p:ID!,$i:ID!,$a:ID){
  updateProjectV2ItemPosition(input:{projectId:$p, itemId:$i, afterId:$a}){ clientMutationId } }' \
  -f p=<projectId> -f i=<itemId> -f a=<afterItemId>
```

## Reading workflows (to see what's enabled)

```bash
gh api graphql -f query='{ viewer { projectV2(number: <num>) {
  workflows(first: 30) { nodes { number name enabled } } } } }' \
  --jq '.data.viewer.projectV2.workflows.nodes[] | "\(if .enabled then "on " else "off" end) \(.name)"'
```

Workflows can be **listed** and **deleted** (`deleteProjectV2Workflow`) but **not created or toggled** via API -- enable them in the UI (`project-setup.md`).

## Project settings and template copy

```bash
gh project edit <num> --owner @me --title "..." --readme "..."          # title / README / visibility
gh project copy <num> --source-owner @me --target-owner @me --title "X" # clones fields + views (template fast-path)
gh project mark-template <num> --owner @me                              # mark as a reusable template
```

## What's NOT in the API

- **Views** -- no create/rename/layout/filter/sort mutation. The Epic + Upcoming views are a one-time UI step (or inherited via `gh project copy`).
- **Workflow toggling** -- enable the native workflows in the UI.

Everything else in the playbooks (`SKILL.md`) is fully scriptable.
