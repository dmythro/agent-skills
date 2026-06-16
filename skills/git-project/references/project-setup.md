# Project Setup and Migration

What's scriptable (`gh`/GraphQL) vs. the small one-time UI step (views + workflow toggles, which have no API).

## 1. Create the project + fields (scripted)

```bash
gh project create --owner @me --title "Roadmap"            # note the new number
# Status already exists (Todo / In Progress / Done). Add Priority:
gh project field-create <num> --owner @me --name "Priority" --data-type SINGLE_SELECT \
  --single-select-options "P0,P1,P2,P3,P4"
# Optional: a coarse phase/stage field
gh project field-create <num> --owner @me --name "Stage" --data-type SINGLE_SELECT \
  --single-select-options "Now,Next,Later,Backlog"
gh label create epic --color B60205 --description "Roadmap epic"
gh project link <num> --owner @me --repo {owner}/{repo}    # surface it in the repo's Projects tab
```

`gh project link` only surfaces the project on the repo -- it does **not** auto-add issues; that's a workflow (step 3).

## 2. The two views (one-time UI -- no API)

Views can't be created, renamed, or laid out via the API. In the project, open the view tabs (`+` to add):

- **Epic** -- rename the default "View 1" to `Epic`. Layout: **Table** (or Board). **Group by: Parent issue** so each epic shows its sub-issue tree. This is the roadmap.
- **Upcoming** -- add a view named `Upcoming`. **Filter:** `-status:Done` (hide finished). **Sort:** `Priority` ascending (so `P0` is on top). Optionally group by `Status`. This is the "what's critical and open" view.

(If you set up many repos under one account, skip this each time by copying a project that already has these views: `gh project copy <template#> --source-owner @me --target-owner @me --title "..."` -- copy clones fields + views.)

## 3. Native workflows (one-time UI toggles -- no API)

Project -> Settings -> **Workflows**. Enable (all ship disabled):

- **Item added to project** -> Set `Status: Todo` -- new items start in Todo.
- **Auto-add sub-issues to project** -- when an epic is on the board, its native children join automatically (pairs with the epic model so you only add the epic).
- **Item closed** -> Set `Status: Done` -- closing an issue advances the board (rule 3; without this, closing does nothing to Status).
- *(optional)* **Pull request merged** -> Set `Status: Done`.
- *(optional)* **Auto-add to project** -- watch `{owner}/{repo}` and add every new matching issue, so you never `item-add` by hand.

These can be listed via the API (`workflows` connection) but only toggled in the UI -- do it once.

## 4. Verify

```bash
gh project field-list <num> --owner @me --format json --jq '[.fields[].name]'   # Status, Priority, Stage...
gh api graphql -f query='{ viewer { projectV2(number: <num>) { views(first:10){nodes{name layout}}
  workflows(first:30){nodes{name enabled}} } } }' \
  --jq '{views:[.data.viewer.projectV2.views.nodes[].name], on:[.data.viewer.projectV2.workflows.nodes[]|select(.enabled)|.name]}'
```

Expect views `["Epic","Upcoming"]` and the workflows above enabled. (Org-owned project: swap `viewer` for `organization(login: "ORG")` and `.data.viewer` for `.data.organization` -- see `cli-and-graphql.md`.)

## Migrating an existing repo's issues

For a repo with loose issues and no structure:

1. **Survey.** `gh issue list --repo {owner}/{repo} --state open --limit 200 --json number,title,labels` and `gh project field-list`/`item-list --format json` to see what exists.
2. **Backfill setup.** Add the `Priority` field, `epic` label, and (if missing) enable the native workflows (steps 1-3).
3. **Define epics.** Create `Epic:`-titled, `epic`-labeled issues for each theme (or relabel existing umbrella issues).
4. **Home every issue.** Link each loose issue under exactly one epic as a native sub-issue (`references/sub-issues.md`); re-parent any that are under the wrong one.
5. **Seed the board.** Add the epics (children auto-add if the workflow is on, else add each); set `Status` from open/closed state and `Priority` per item (`references/cli-and-graphql.md`).
6. **Confirm** with the verify query above, then eyeball the Epic and Upcoming views.

Do the sub-issue links in chunks of ~25 (the batch limit).
