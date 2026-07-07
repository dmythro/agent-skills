# Issue Types and Milestones

Native issue metadata that replaces label conventions: **Type** classifies the kind of work (Task/Bug/Feature), **Milestone** scopes it to a dated deliverable. Both live on the **issue** (not the project) and are mirrored onto the board as built-in columns for grouping/filtering -- `gh project item-edit` cannot set them and they never appear in `field-list`. Requires `gh` >= 2.94 for the native flags.

## Issue Types

Issue types are defined at the **organization** level (Settings -> Planning -> Issue types) and apply to every repo in the org. Defaults: `Task`, `Bug`, `Feature`; orgs can add custom types (e.g. `Epic`). **Personal (user-owned) repos have no issue types** -- `--type` there fails with `type "..." not found; available types:` (empty list); fall back to labels.

### Set / change / remove

```bash
gh issue create --title "..." --type Bug --body "..."
gh issue edit <n> --type Feature
gh issue edit <n> --remove-type
```

Types are matched by name, case-insensitive. One type per issue.

### Query and filter

```bash
gh issue list --type Task --json number,title             # filter by type
gh api orgs/{org}/issue-types                             # list the org's types (read-only)
gh api repos/{owner}/{repo}/issues/<n> --jq '.type.name'  # a single issue's type
```

GraphQL: `issueType { name }` on `Issue`; the org's catalog is `organization(login: "ORG") { issueTypes(first: 20) { nodes { name isEnabled } } }`.

Managing the catalog itself (create/update/delete a type) is org-admin REST: `POST/PUT/DELETE /orgs/{org}/issue-types[/{id}]`.

### Types on the board

In any project view, `Type` is available as a column, and works in **group by**, **slice by**, and filters (`type:Bug`). This is what replaces `bug`/`feature`/`task` labels: zero label noise, consistent colors, and the grouping is native. With a custom `Epic` type, even the `epic` label can go.

## Milestones

A milestone is a repo-scoped, dated, closeable bucket -- the "when does this scope ship" axis, complementary to epics ("what theme is this"). An issue typically has both a parent epic and a milestone. Closing a milestone is the explicit end of that scope.

There is **no `gh milestone` subcommand** -- issue-side flags cover assignment; milestone CRUD is REST.

### CRUD

```bash
# list open (default); ?state=all / ?state=closed for the rest
gh api repos/{owner}/{repo}/milestones --jq '[.[] | {number, title, due_on, open_issues, closed_issues}]'
# fetch one by NUMBER
gh api repos/{owner}/{repo}/milestones/<N>
# create -- due_on is ISO 8601; GitHub normalizes it to the repo timezone's end-of-day
gh api --method POST repos/{owner}/{repo}/milestones -f title="v1.0" -f due_on="2026-08-01T00:00:00Z" -f description="<scope>"
# edit / close / reopen
gh api --method PATCH repos/{owner}/{repo}/milestones/<N> -f due_on="2026-09-01T00:00:00Z"
gh api --method PATCH repos/{owner}/{repo}/milestones/<N> -f state=closed
# delete (issues lose the association; usually prefer closing)
gh api --method DELETE repos/{owner}/{repo}/milestones/<N>
```

### Assign issues and PRs

```bash
gh issue create --title "..." --milestone "v1.0"    # by TITLE
gh issue edit <n> --milestone "v1.0"
gh issue edit <n> --remove-milestone
gh pr create --title "..." --milestone "v1.0"       # PRs take milestones too
gh issue list --milestone "v1.0"                    # -m accepts a title or the milestone NUMBER
```

Note: a just-assigned milestone can take a few seconds to show up in `gh issue list -m` results (search-index lag) -- don't treat an immediate empty list as failure.

### The conventions

- **"Current milestone"** = the **open** milestone with the **smallest due date**. A passed due date still counts -- overdue is the most urgent scope, not a skipped one. Milestones without a due date only qualify when nothing dated is open:

  ```bash
  gh api repos/{owner}/{repo}/milestones --jq 'sort_by(.due_on // "9999-12-31") | first | {number, title, due_on}'
  ```

  (The endpoint returns open milestones by default; `// "9999-12-31"` sorts undated ones last. Returns `null` when no milestone is open.)

- **"Milestone N"** = the milestone with **number N** -- the stable identifier in the `/milestone/N` URL and the REST `number` field. It is *not* "the Nth open milestone" or a list position. Resolve it directly: `gh api repos/{owner}/{repo}/milestones/<N>`.

### Close out a scope

When the work in a milestone is done, close the milestone itself -- that is the signal the scope shipped:

```bash
gh api repos/{owner}/{repo}/milestones/<N> --jq '{title, open_issues}'   # expect open_issues: 0
gh api --method PATCH repos/{owner}/{repo}/milestones/<N> -f state=closed
```

If `open_issues` > 0, list the stragglers first: `gh issue list --milestone <N> --json number,title,state`. Move or close them deliberately -- never close a milestone over open issues silently.

### Milestones on the board

`Milestone` is a built-in project column: group, slice, or filter by it (`milestone:"v1.0"`). A milestone-grouped view is the release board; combined with the Epic view you get both axes without any extra project fields.

## Key Gotchas

1. **Issue types are org-only** -- on personal repos `--type` errors with an empty "available types" list; use labels there.
2. **Set on the issue, not the item** -- `item-edit` has no Type/Milestone; the board columns are read-through mirrors.
3. **Milestones are per-repo** -- a multi-repo project shows them all in one column, but a milestone can't span repos (unlike the project's own fields).
4. **`--milestone` matches by title, `milestones/<N>` by number** -- and "milestone N" in conversation means number N, not the Nth open one.
5. **Current milestone includes overdue** -- smallest due date among open, passed or not; don't filter out past dates.
