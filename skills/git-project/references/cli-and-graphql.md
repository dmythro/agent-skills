# gh project + GraphQL Reference

Verified against `gh` v2.102.0 (the pin in `SKILL.md`). Projects are addressed by **number** under an `--owner` (`@me` or an org login). `gh project` covers projects, fields, and items; **views and workflows have no create/update API** (UI-only -- see `project-setup.md`). Issue-side metadata (types, milestones, sub-issue links) is set with `gh issue` flags, not `gh project` -- see `types-and-milestones.md` and `sub-issues.md`.

**Owner / org note:** the `--owner @me` examples below take an **org login** instead for org-owned projects. The GraphQL **project reads** here use `viewer { projectV2 }` (the `@me` case); for an org-owned project replace it with `organization(login: "ORG") { projectV2 }` (and `.data.viewer` -> `.data.organization` in the `--jq`). Repo-scoped reads (`repository(owner, name)`, REST `repos/{owner}/{repo}/...`) are unaffected.

## Find projects and their IDs

```bash
gh project list --owner @me                        # number, title, ID (PVT_...)
gh project view <num> --owner @me                  # human summary
gh project view <num> --owner @me --format json --jq '{id, title, number}'   # the PVT_ project id
```

## Fields and their option IDs

Only the ID form of `item-edit` (below) needs these. Name-based edits resolve fields and options themselves:

```bash
gh project field-list <num> --owner @me --format json \
  --jq '.fields[] | select(.type=="ProjectV2SingleSelectField") | {name, id, options: [.options[] | {name, id}]}'
```

Create a single-select field:

```bash
gh project field-create <num> --owner @me --name "Priority" \
  --data-type SINGLE_SELECT --single-select-options "P0,P1,P2,P3"
```

`--data-type` is one of `TEXT | SINGLE_SELECT | DATE | NUMBER`. (A project ships with a built-in `Status` field already populated `Todo/In Progress/Done`.)

## Items (issues on the board)

```bash
gh project item-add <num> --owner @me --url <issue-url>            # add an issue/PR
gh project item-list <num> --owner @me --format json \
  --jq '.items[] | {id, title, content: .content.number, status, priority}'   # item ids (PVTI_...)
gh project item-list <num> --owner @me --query "-status:Done is:open" --format json   # Projects filter syntax
gh project item-list <num> --owner @me --field Status --field Priority   # extra TABLE columns only
```

The JSON carries each project field's value under its **lowercased** name (`status`, `priority`), plus `milestone` (`{title, dueOn, description}`), `labels`, `repository` and `content` (`type`, `number`, `url`). `--field` adds table columns only: combined with `--format` it fails with `cannot use --format with --field or --field-id`.

### Edit by name (gh >= 2.97 -- the default)

The project is picked by number and `--owner`, the item by the **issue or PR URL** (not a project URL), the field by name, and a single-select option by its name in `--value`. For non-draft issues, each call sets one field. A wrong name fails before writing and lists the valid ones (`field "X" not found in project; available fields: ...`, `option "P9" not found on field "Priority"; available options: P0, P1, P2, P3`), so the names need no lookup first:

```bash
gh project item-edit <num> --owner @me --url <issue-url> --field Status --value "In Progress"
gh project item-edit <num> --owner @me --url <issue-url> --field Priority --value P1
gh project item-edit <num> --owner @me --url <issue-url> --field Estimate --number 3
gh project item-edit <num> --owner @me --url <issue-url> --field Due --date 2026-07-01
gh project item-edit <num> --owner @me --url <issue-url> --field Priority --clear          # unset
```

### Edit by ID (older gh, scripted loops)

The ID form needs the **item id** (`PVTI_...`), the **project id** (`PVT_...`), the **field id**, and -- for single-selects -- the **option id**. It saves the name lookups on every call, which matters in a loop over many items:

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

### Resolve an issue's item id (ID form only)

gh's `--jq` is built in (it is **not** the standalone `jq`, so no `--arg`) -- interpolate the issue number directly, or pipe to real `jq`:

```bash
gh project item-list <num> --owner @me --format json \
  --jq '.items[] | select(.content.number==<issueNumber>) | .id'
```

### Read items with their issue metadata (Type, Milestone, parent)

`item-list` gives the milestone but not the issue type or the parent epic -- read those through the items' content in GraphQL (org-owned: swap `viewer` per the note above):

```bash
gh api graphql -f query='{ viewer { projectV2(number: <num>) { items(first: 50) { nodes { id content { ... on Issue { number title state issueType { name } milestone { title number dueOn } parent { number } } } } } } } }' \
  --jq '[.data.viewer.projectV2.items.nodes[] | {item: .id, issue: .content.number, type: .content.issueType.name, milestone: .content.milestone.title, epic: .content.parent.number}]'
```

`issueType` is null on personal-repo issues (types are org-only), `milestone`/`parent` are null when unset. These are **not** project fields: `item-edit` cannot set them (set them on the issue -- `types-and-milestones.md`).

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
