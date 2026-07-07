---
name: git-project
description: >-
  GitHub Projects (v2) setup and management via gh CLI + GraphQL: organize a
  repo's issues into epics with native sub-issues, drive a board Status flow
  (Todo -> In Progress -> Done), set Project Priority, classify with native
  issue Types (Task/Bug/Feature) instead of labels, scope work with milestones
  (incl. resolving "current milestone"), and maintain an Epic + Upcoming
  roadmap. Use when setting up or managing a GitHub Project/roadmap, creating
  an epic with issues, adding or moving issues between epics, picking up or
  closing work on the board, (re)prioritizing, setting issue types, creating
  or closing milestones, enabling project workflows, or configuring gh project
  read-only allowlists. Not for PR workflows (git-pr), CI/CD status (git-ci),
  or commit messages (git-commit)
---

# GitHub Project Management

**Primary skill for organizing a repository's issues into an epic-based roadmap on a GitHub Project (v2) and running the day-to-day on it.** Covers setup (project, fields, views, native workflows) and operations (create epics with sub-issues, drive Status, move work, prioritize). GitHub-only, via `gh project` + `gh api` (REST sub-issues + GraphQL).

The conventions here are non-obvious and easy to get half-right: the UI nests issues by **native parent/child links** and tracks progress on a **separate board Status field** -- neither of which a markdown checklist or a closed issue touches. Get those two wrong and the work *looks* done while the structure silently drifts. This skill makes the flow correct-by-default.

## When to Use

- **Setting up a project/roadmap** -- create the Project, fields (Status, Priority), the Epic + Upcoming views, enable native workflows
- **Creating an epic with issues** -- "make an epic X with these issues", "group these under an epic"
- **Filing / moving work** -- attach an issue under an epic, **move an issue between epics**, add new work
- **Driving the board** -- pick up an issue (-> In Progress), close one (-> Done), (re)prioritize
- **Classifying work** -- set native issue Types (Task/Bug/Feature) instead of type labels
- **Milestone scoping** -- create a milestone for a deliverable, assign issues, close it out; resolve "current milestone" / "milestone N"
- **Migrating** an existing repo's loose issues into this structure
- **Configuring tool allowlists** -- auto-approval patterns for read-only `gh project` commands

## Critical Rules

These are the traps -- each is a place where the obvious action leaves the structure wrong.

1. **Epics nest by native sub-issues, NOT markdown checklists.** A `- [ ] #123` bullet is cosmetic; it does not create the parent/child link the UI and roadmap read. Link natively: `gh issue edit <epic> --add-sub-issue <child>` / `gh issue create --parent <epic>` (gh >= 2.94), or the REST sub-issues API. See `references/sub-issues.md`.
2. **Moving an issue between epics = re-parent the native link.** One command: `gh issue edit <child> --parent <newEpic>` (replaces the old parent), then tidy body text. Editing bullets alone leaves it under the old epic -- the most common "looks moved but isn't" miss.
3. **The board Status is not automatic.** Closing an issue does not set `Done` unless the native *Item closed* workflow is enabled. Set Status explicitly, or enable the native workflows once (see `references/project-setup.md`).
4. **Single-selects (Status, Priority) are set by option ID, not name.** Look up field IDs + option IDs first (`gh project field-list <num> --owner @me --format json`), then `item-edit`. Names silently no-op.
5. **Views and workflow-enabling are one-time UI; everything else is scripted.** There is no API to create/rename a view or toggle a workflow. Do those once in the UI (or copy a template); script the rest.
6. **Type and Milestone are issue metadata, NOT project fields.** Set them on the issue (`gh issue edit <n> --type Bug --milestone "v1.0"`); the board mirrors them as built-in columns for grouping/filtering, but `gh project item-edit` cannot touch them and they never appear in `field-list`. Issue types exist only in **org** repos. See `references/types-and-milestones.md`.

## Prerequisites

- **GitHub only** -- Projects v2 has no GitLab equivalent.
- **`gh` >= 2.94** for the native issue flags used here (`--type`, `--milestone`, `--parent`, `--add-sub-issue`); older `gh` falls back to the REST recipes in the references.
- **Token scopes gate what works.** `gh auth status` shows them. `gh project` commands and any `projectV2` GraphQL need the `project` scope (`gh auth refresh -s project`); `read:project` alone is enough for read-only queries. Without the scope these fail with auth/permission errors even though everything `repo`-scoped works -- the classic "project features seem missing" trap. Issues, sub-issues, types, and milestones ride on the ordinary `repo` scope; org-owned project access also relies on `read:org`.
- Projects are addressed by **number** under an `--owner` (`@me` or an org login). The repo and project may have different owners. For an **org-owned** project, pass the org login to `--owner`, and in GraphQL reads use `organization(login: "ORG") { projectV2 }` instead of `viewer { projectV2 }` (see `references/cli-and-graphql.md`).

## The Model

| Layer | What it is | Mechanism |
|-------|-----------|-----------|
| **Epic** | A big work item -- an issue titled `Epic: ...`, labeled `epic` | `gh issue create` + the `epic` label |
| **Sub-issue** | A unit of work homed under exactly one epic | native parent/child link (`--parent` / `--add-sub-issue`) |
| **Type** | What kind of work: Task / Bug / Feature (org repos only) | native **issue type** (`gh issue edit --type`), not labels |
| **Milestone** | A dated, closeable scope -- a release or delivery slice, often cross-epic | repo milestone (`gh issue edit --milestone`); mirrored on the board |
| **Board state** | Where each item sits + its priority | Project fields: **Status** (Todo/In Progress/Done), **Priority** (single-select) |

Every issue is homed under an epic (or is one). **Priority lives on the Project `Priority` field** (set via `item-edit`) and **kind lives on the native issue Type** -- neither is a label. That keeps label noise near zero: the only label in play is `epic` (identifies epics for filtering/grouping; if the org defines a custom `Epic` issue type, prefer that and drop even this label). Epics answer "what theme does this belong to"; milestones answer "when does this scope ship" -- an issue typically has both. The **Epic** view shows the tree; the **Upcoming** view filters out Done and sorts by Priority.

## Operations (Playbooks)

Concrete sequences. `{owner}/{repo}` are filled by gh from the current repo; `<num>` is the project number. ID-discovery details are in `references/cli-and-graphql.md`.

### Create an epic with issues

```bash
# Epic issue (gh issue create prints the URL; the number is its last path segment)
epic=$(gh issue create --title "Epic: <name>" --label epic --body "<goal>"); epic=${epic##*/}
# Link existing issues as NATIVE sub-issues (flag repeats; on a partial GraphQL failure re-run -- it's a
# transient sub-issue burst limit, and already-linked children are unaffected)
gh issue edit $epic --add-sub-issue <existing#> --add-sub-issue <existing#>
# New work goes straight under the epic, typed and scoped at creation
# (--type is org-only: DROP it on personal repos or the whole command fails with 'type not found')
gh issue create --title "<task>" --type Task --milestone "<title>" --parent $epic --body "<detail>"
# Put the epic on the board; its children auto-join IF the native "Auto-add sub-issues" workflow is on
gh project item-add <num> --owner @me --url "$(gh issue view $epic --json url --jq .url)"
# (if that workflow is off, item-add each child's URL too)
```

### Add an issue to an existing epic

```bash
gh issue edit <epic> --add-sub-issue <child>     # by number or URL; also re-parents if homed elsewhere
```

### Move an issue between epics (re-parent -- rule 2)

```bash
gh issue edit <child> --parent <newEpic>         # replaces the old parent in one step
# then tidy any body bullets that referenced the old epic
```

### Classify and scope (Type + Milestone -- rule 6)

```bash
gh issue edit <n> --type Bug                     # org repos only; Task/Bug/Feature (+ org customs)
gh issue edit <n> --milestone "v1.0"             # by TITLE; --remove-milestone / --remove-type to unset
gh issue list --milestone "v1.0" --json number,title,state   # -m takes a title or the milestone NUMBER
```

### Milestones (create, resolve "current", close out a scope)

"**Current milestone**" = the open milestone with the **smallest due date** -- past-due included, that's the most urgent one. "**Milestone N**" = the milestone with **number N** (the `/milestone/N` URL segment), never "the Nth open one".

```bash
# create (no gh subcommand -- REST)
gh api --method POST repos/{owner}/{repo}/milestones -f title="v1.0" -f due_on="2026-08-01T00:00:00Z" -f description="<scope>"
# current milestone (open only; undated ones sort last)
gh api repos/{owner}/{repo}/milestones --jq 'sort_by(.due_on // "9999-12-31") | first | {number, title, due_on}'
# close out the scope once nothing is left open in it
gh api repos/{owner}/{repo}/milestones/<N> --jq '{title, open_issues}'      # expect open_issues: 0
gh api --method PATCH repos/{owner}/{repo}/milestones/<N> -f state=closed
```

### Pick up / finish / prioritize (board Status + Priority)

```bash
# Discover the item id + field/option ids once (see references/cli-and-graphql.md), then:
# pick up
gh project item-edit --id <item> --project-id <proj> --field-id <statusField> --single-select-option-id <inProgress>
# finish -- close the issue (pick ONE close form; the plain one means "completed"):
gh issue close <issue>                              # done as planned
gh issue close <issue> --reason "not planned"       # abandoned
gh issue close <issue> --duplicate-of <original>    # duplicate; links it natively to the original (gh >= 2.88)
# closing normally advances the board via the native "Item closed" workflow -- nothing more to do.
# ONLY IF that workflow is off, set Status manually:
gh project item-edit --id <item> --project-id <proj> --field-id <statusField> --single-select-option-id <done>
# (re)prioritize
gh project item-edit --id <item> --project-id <proj> --field-id <priorityField> --single-select-option-id <p1>
```

> **Reference**: `references/cli-and-graphql.md` -- full command set, getting `<item>`/`<proj>`/field+option ids (`field-list`/`item-list --format json`), and `updateProjectV2ItemPosition` for roadmap ordering. `references/sub-issues.md` -- native link flags, the REST fallback (database-id requirement), re-parenting, and the ~25/request batch limit. `references/types-and-milestones.md` -- org issue types, milestone CRUD, and the current-milestone / milestone-N conventions in full.

## Setup (one-time)

> **Reference**: `references/project-setup.md` -- end to end: create the Project, add `Status`/`Priority` (and optional `Stage`) fields, build the **Epic** and **Upcoming** views (UI -- no API), enable the native workflows (*Item added*, *Auto-add sub-issues*, *Item closed -> Done*), the optional template-copy fast-path, and the **migration** playbook for an existing repo.

Quick shape:
```bash
gh project create --owner @me --title "Roadmap"
gh project field-create <num> --owner @me --name "Priority" --data-type SINGLE_SELECT \
  --single-select-options "P0,P1,P2,P3"
gh label create epic --color B60205 --description "Roadmap epic"
```
Then, once in the UI: the Epic + Upcoming views and Settings -> Workflows toggles (these have no API).

## Read-Only vs Write Classification

- **Read-only** (safe to auto-approve): `gh project list/view/field-list/item-list`, GraphQL read queries, `gh label list`, `gh issue list/view`, milestone reads (`gh api repos/{owner}/{repo}/milestones`), issue-type reads (`gh api orgs/{org}/issue-types`)
- **Write** (require approval): `gh project create/copy/edit/link/field-create/item-add/item-edit/item-archive/item-delete`, `gh label create`, `gh issue create/edit` (incl. `--type/--milestone/--parent/--add-sub-issue`), milestone `POST`/`PATCH`/`DELETE`, sub-issue `POST`/`DELETE`, GraphQL mutations

> **Reference**: See `references/allowlist.md` for read-only `gh project` patterns and the opt-in write set.

## Key Gotchas

1. **Native links, not checklists** -- the tree is built from sub-issue links; bullets are decoration (rule 1).
2. **Re-parent to move** -- `gh issue edit <child> --parent <newEpic>`; body edits alone don't move it (rule 2).
3. **REST sub-issues take the database `id`** -- if you drop to `gh api .../sub_issues`, fetch it with `--jq .id`; the issue *number* won't work (the native `gh` flags take numbers/URLs).
4. **Single-selects set by option id** -- discover ids with `field-list --format json` before `item-edit`.
5. **Views + workflow toggles are UI-only** -- no API; do them once (or copy a template) (rule 5).
6. **Sub-issue mutations batch ~25/request** -- larger batches hit `RESOURCE_LIMITS_EXCEEDED`; a multi-`--add-sub-issue` edit can partially fail on the same limit -- re-running is safe.
7. **Type/Milestone are not project fields** -- set on the issue, mirrored on the board; `item-edit` can't set them and `field-list` won't show them (rule 6). Types are org-only (`--type` on a personal repo: `type "..." not found; available types:` -- fall back to labels there).
8. **"Current milestone" = smallest due date among open** (past-due included); **"milestone N" = number N**, not the Nth open (see `references/types-and-milestones.md`).
