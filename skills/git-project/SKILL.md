---
name: git-project
description: >-
  GitHub Projects (v2) setup and management via gh CLI + GraphQL: organize a
  repo's issues into epics with native sub-issues, drive a board Status flow
  (Todo -> In Progress -> Done), set Project Priority, and maintain an Epic +
  Upcoming roadmap. Use when setting up or managing a GitHub Project/roadmap,
  creating an epic with issues, adding or moving issues between epics, picking
  up or closing work on the board, (re)prioritizing, enabling project
  workflows, or configuring gh project read-only allowlists. Not for PR
  workflows (git-pr), CI/CD status (git-ci), or commit messages (git-commit)
---

# GitHub Project Management

**Primary skill for organizing a repository's issues into an epic-based roadmap on a GitHub Project (v2) and running the day-to-day on it.** Covers setup (project, fields, views, native workflows) and operations (create epics with sub-issues, drive Status, move work, prioritize). GitHub-only, via `gh project` + `gh api` (REST sub-issues + GraphQL).

The conventions here are non-obvious and easy to get half-right: the UI nests issues by **native parent/child links** and tracks progress on a **separate board Status field** -- neither of which a markdown checklist or a closed issue touches. Get those two wrong and the work *looks* done while the structure silently drifts. This skill makes the flow correct-by-default.

## When to Use

- **Setting up a project/roadmap** -- create the Project, fields (Status, Priority), the Epic + Upcoming views, enable native workflows
- **Creating an epic with issues** -- "make an epic X with these issues", "group these under an epic"
- **Filing / moving work** -- attach an issue under an epic, **move an issue between epics**, add new work
- **Driving the board** -- pick up an issue (-> In Progress), close one (-> Done), (re)prioritize
- **Migrating** an existing repo's loose issues into this structure
- **Configuring tool allowlists** -- auto-approval patterns for read-only `gh project` commands

## Critical Rules

These are the traps -- each is a place where the obvious action leaves the structure wrong.

1. **Epics nest by native sub-issues, NOT markdown checklists.** A `- [ ] #123` bullet is cosmetic; it does not create the parent/child link the UI and roadmap read. Use the sub-issues API (`POST .../issues/{parent}/sub_issues`). See `references/sub-issues.md`.
2. **Moving an issue between epics = re-parent the native link.** Remove the old link and add the new one (`DELETE .../issues/{old}/sub_issue` + `POST .../issues/{new}/sub_issues`), then tidy body text. Editing bullets alone leaves it under the old epic -- the most common "looks moved but isn't" miss.
3. **The board Status is not automatic.** Closing an issue does not set `Done` unless the native *Item closed* workflow is enabled. Set Status explicitly, or enable the native workflows once (see `references/project-setup.md`).
4. **Single-selects (Status, Priority) are set by option ID, not name.** Look up field IDs + option IDs first (`gh project field-list --format json`), then `item-edit`. Names silently no-op.
5. **Views and workflow-enabling are one-time UI; everything else is scripted.** There is no API to create/rename a view or toggle a workflow. Do those once in the UI (or copy a template); script the rest.

## Prerequisites

- **GitHub only** -- Projects v2 has no GitLab equivalent.
- **`gh` with the `project` scope** -- `gh auth status` must list `project`; else `gh auth refresh -s project`.
- Projects are addressed by **number** under an `--owner` (`@me` or an org). The repo and project may have different owners.

## The Model

| Layer | What it is | Mechanism |
|-------|-----------|-----------|
| **Epic** | A big work item -- an issue titled `Epic: ...`, labeled `epic` | `gh issue create` + the `epic` label |
| **Sub-issue** | A unit of work homed under exactly one epic | native parent/child link (sub-issues API) |
| **Board state** | Where each item sits + its priority | Project fields: **Status** (Todo/In Progress/Done), **Priority** (single-select) |

Every issue is homed under an epic (or is one). **Priority lives on the Project `Priority` field** (set via `item-edit`), not on labels -- the only label in play is `epic` (it identifies epics for filtering/grouping). The **Epic** view shows the tree; the **Upcoming** view filters out Done and sorts by Priority.

## Operations (Playbooks)

Concrete sequences. `{owner}/{repo}` are filled by gh from the current repo; `<num>` is the project number. ID-discovery details are in `references/cli-and-graphql.md`.

### Create an epic with issues

```bash
# Epic issue (gh issue create prints the URL; the number is its last path segment)
epic=$(gh issue create --title "Epic: <name>" --label epic --body "<goal>"); epic=${epic##*/}
# Each child -> link as a NATIVE sub-issue, by the child's DATABASE id (not its number)
for child in <existing#> <existing#>; do
  gh api --method POST repos/{owner}/{repo}/issues/$epic/sub_issues \
    -F sub_issue_id="$(gh api repos/{owner}/{repo}/issues/$child --jq .id)"
done
# Put the epic on the board; its children auto-join IF the native "Auto-add sub-issues" workflow is on
gh project item-add <num> --owner @me --url "$(gh issue view $epic --json url --jq .url)"
# (if that workflow is off, item-add each child's URL too)
```

### Add an issue to an existing epic

```bash
gh api --method POST repos/{owner}/{repo}/issues/<epic>/sub_issues \
  -F sub_issue_id="$(gh api repos/{owner}/{repo}/issues/<child> --jq .id)"
```

### Move an issue between epics (re-parent -- rule 2)

```bash
sid="$(gh api repos/{owner}/{repo}/issues/<child> --jq .id)"
gh api --method DELETE repos/{owner}/{repo}/issues/<oldEpic>/sub_issue  -F sub_issue_id="$sid"
gh api --method POST   repos/{owner}/{repo}/issues/<newEpic>/sub_issues -F sub_issue_id="$sid"
# then tidy any body bullets that referenced the old epic
```

### Pick up / finish / prioritize (board Status + Priority)

```bash
# Discover the item id + field/option ids once (see references/cli-and-graphql.md), then:
# pick up
gh project item-edit --id <item> --project-id <proj> --field-id <statusField> --single-select-option-id <inProgress>
# finish -- close the issue; the native "Item closed" workflow sets Status: Done (set it here only if that workflow is off)
gh issue close <issue>
gh project item-edit --id <item> --project-id <proj> --field-id <statusField> --single-select-option-id <done>
# (re)prioritize
gh project item-edit --id <item> --project-id <proj> --field-id <priorityField> --single-select-option-id <p1>
```

> **Reference**: `references/cli-and-graphql.md` -- full command set, getting `<item>`/`<proj>`/field+option ids (`field-list`/`item-list --format json`), and `updateProjectV2ItemPosition` for roadmap ordering. `references/sub-issues.md` -- the native link API, the database-id requirement, re-parenting, and the ~25/request batch limit.

## Setup (one-time)

> **Reference**: `references/project-setup.md` -- end to end: create the Project, add `Status`/`Priority` (and optional `Stage`) fields, build the **Epic** and **Upcoming** views (UI -- no API), enable the native workflows (*Item added*, *Auto-add sub-issues*, *Item closed -> Done*), the optional template-copy fast-path, and the **migration** playbook for an existing repo.

Quick shape:
```bash
gh project create --owner @me --title "Roadmap"
gh project field-create <num> --owner @me --name "Priority" --data-type SINGLE_SELECT \
  --single-select-options "P0,P1,P2,P3,P4"
gh label create epic --color B60205 --description "Roadmap epic"
```
Then, once in the UI: the Epic + Upcoming views and Settings -> Workflows toggles (these have no API).

## Read-Only vs Write Classification

- **Read-only** (safe to auto-approve): `gh project list/view/field-list/item-list`, GraphQL read queries, `gh label list`, `gh issue list/view`
- **Write** (require approval): `gh project create/copy/edit/field-create/item-add/item-edit/item-archive`, `gh label create`, `gh issue create/edit`, sub-issue `POST`/`DELETE`, GraphQL mutations

> **Reference**: See `references/allowlist.md` for read-only `gh project` patterns and the opt-in write set.

## Key Gotchas

1. **Native links, not checklists** -- the tree is built from sub-issue links; bullets are decoration (rule 1).
2. **Re-parent to move** -- DELETE old + POST new; body edits alone don't move it (rule 2).
3. **`sub_issue_id` is the database `id`** -- fetch it with `gh api .../issues/{n} --jq .id`; the issue *number* won't work.
4. **Single-selects set by option id** -- discover ids with `field-list --format json` before `item-edit`.
5. **Views + workflow toggles are UI-only** -- no API; do them once (or copy a template) (rule 5).
6. **Sub-issue mutations batch ~25/request** -- larger batches hit `RESOURCE_LIMITS_EXCEEDED`; chunk them.
