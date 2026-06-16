# Native Sub-Issues (Parent / Child)

GitHub's roadmap tree is built from **native sub-issue links**, not markdown checklists. A `- [ ] #123` bullet renders a box but creates no structural link -- the Project, the issue's "Sub-issues" panel, and `Sub-issues progress` all read the native parent/child relationship. This is the single most common "looks done but the link was never made" miss.

## The database-id requirement

The sub-issues REST API identifies the child by its **database `id`** (a large integer like `4573108877`), **not** its issue number and **not** its `node_id`. Always resolve it first:

```bash
gh api repos/{owner}/{repo}/issues/<number> --jq .id     # -> 4573108877
```

## List a parent's children

```bash
gh api repos/{owner}/{repo}/issues/<epic>/sub_issues --jq '.[] | {number, id, state, title}'
```

Returns each child's `number` and `id` -- the `id` is what you pass to re-parent.

## Add a child (link)

```bash
gh api --method POST repos/{owner}/{repo}/issues/<parent>/sub_issues \
  -F sub_issue_id="$(gh api repos/{owner}/{repo}/issues/<child> --jq .id)"
```

`-F` (not `-f`) sends `sub_issue_id` as a number. The child must not already have a parent -- re-parent instead (below).

## Remove a child (unlink)

Note the **singular** `sub_issue` on DELETE (vs. plural `sub_issues` on POST/GET):

```bash
gh api --method DELETE repos/{owner}/{repo}/issues/<parent>/sub_issue \
  -F sub_issue_id="$(gh api repos/{owner}/{repo}/issues/<child> --jq .id)"
```

## Move between epics = re-parent (the trap)

Moving an issue is **not** a body edit -- it is unlink-from-old + link-to-new. Editing checklist bullets leaves the native parent unchanged, so the UI still nests it under the old epic:

```bash
sid="$(gh api repos/{owner}/{repo}/issues/<child> --jq .id)"
gh api --method DELETE repos/{owner}/{repo}/issues/<oldEpic>/sub_issue  -F sub_issue_id="$sid"
gh api --method POST   repos/{owner}/{repo}/issues/<newEpic>/sub_issues -F sub_issue_id="$sid"
# only now tidy any body bullets that referenced the old epic
```

## Find an issue's current parent

The issue REST object does **not** expose its parent -- query it with GraphQL:

```bash
gh api graphql -f query='{ repository(owner:"{owner}",name:"{repo}"){ issue(number:<child>){ parent { number title } } } }' \
  --jq '.data.repository.issue.parent.number // "none"'
```

## GraphQL alternative

The REST endpoints above are simplest. The GraphQL equivalents are `addSubIssue` / `removeSubIssue` mutations (they take node IDs, not database ids):

```bash
gh api graphql -f query='mutation($p:ID!,$c:ID!){ addSubIssue(input:{issueId:$p, subIssueId:$c}){ issue { number } } }' \
  -f p=<parent-node-id> -f c=<child-node-id>
```

## Batch limit

Sub-issue mutations are rate/size limited to roughly **25 per request/burst** -- larger batches return `RESOURCE_LIMITS_EXCEEDED`. When wiring up a big epic, chunk the links into groups of ~25 and let each settle.

## Why this matters

Native links drive three things a checklist cannot: the Project's nesting/grouping, the `Sub-issues progress` bar, and roadmap rollups. Keep every issue homed under exactly one epic via these links, and the board and roadmap stay correct without manual bookkeeping.
