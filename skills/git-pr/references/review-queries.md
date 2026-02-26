# Review State Queries

Advanced patterns for querying PR/MR review state across GitHub and GitLab.

## GitHub: Latest Review Per Reviewer

Each reviewer can submit multiple reviews. Get the most recent state per reviewer:

```bash
gh pr view {number} --json reviews --jq '
  .reviews
  | sort_by([.author.login, .submittedAt])
  | group_by(.author.login)
  | map({
      user: .[0].author.login,
      latest: .[-1].state,
      count: length
    })
  | sort_by(.user)'
```

States: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, `PENDING`, `DISMISSED`.

## GitHub: Quick Approval Check

```bash
# Is the PR approved by at least one reviewer?
gh pr view {number} --json reviews --jq '
  [.reviews[] | select(.state == "APPROVED")] | length > 0'

# Who approved?
gh pr view {number} --json reviews --jq '
  [.reviews[] | select(.state == "APPROVED") | .author.login] | unique'

# Who requested changes?
gh pr view {number} --json reviews --jq '
  [.reviews[] | select(.state == "CHANGES_REQUESTED") | .author.login] | unique'
```

## GitHub: Pending Reviewers

```bash
# Requested but haven't reviewed yet
gh pr view {number} --json reviewRequests --jq '.reviewRequests[].login'

# Or with type info (user vs team)
gh pr view {number} --json reviewRequests --jq '.reviewRequests[] | {login, type: .type}'
```

## GitLab: Approval State

```bash
# Basic upvote info from MR view (emoji/thumbs, not formal approvals)
glab mr view {iid} -F json | jq '{upvotes:.upvotes,reviewers:[.reviewers[]?.username]}'

# Detailed approval rules and who approved
glab api projects/{project_id}/merge_requests/{iid}/approvals | jq '{approved:.approved,approvals_required:.approvals_required,approvals_left:.approvals_left,approvers:[.approved_by[]?.user.username]}'

# Approval rules (required approvers per rule)
glab api projects/{project_id}/merge_requests/{iid}/approval_rules | jq '[.[] | {name:.name,approvals_required:.approvals_required,approvers:[.eligible_approvers[]?.username]}]'
```

## PR/MR Metadata

### Labels and Milestones

**GitHub:**
```bash
gh pr view {number} --json labels --jq '[.labels[].name] | join(", ")'
gh pr view {number} --json milestone --jq '.milestone.title'
```

**GitLab:**
```bash
glab mr view {iid} -F json | jq '{labels:.labels,milestone:.milestone.title}'
```

### Linked Issues

**GitHub:**
```bash
gh pr view {number} --json closingIssuesReferences --jq '
  .closingIssuesReferences[] | "#\(.number) \(.title)"'
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/closes_issues | jq '[.[] | {iid:.iid,title:.title}]'
```

### PR/MR Age and Activity

**GitHub:**
```bash
gh pr view {number} --json createdAt,updatedAt,author --jq '
  "Created: \(.createdAt)\nUpdated: \(.updatedAt)\nAuthor: \(.author.login)"'
```

**GitLab:**
```bash
glab mr view {iid} -F json | jq '{created:.created_at,updated:.updated_at,author:.author.username}'
```

## Batch Queries

### All Open PRs/MRs Needing Review

**GitHub:**
```bash
gh pr list --search "review-requested:@me" --json number,title,url
```

**GitLab:**
```bash
glab mr list --reviewer=@me -F json | jq '[.[] | {iid:.iid,title:.title,url:.web_url}]'
```

### PRs with Failing CI (GitHub)

```bash
gh pr list --json number,title,statusCheckRollup --jq '
  .[] | select(.statusCheckRollup | any(.conclusion == "FAILURE"))
  | "\(.number) \(.title)"'
```

### Stale PRs/MRs (No Activity in 7+ Days)

**GitHub:**
```bash
gh pr list --json number,title,updatedAt --jq '
  .[] | select(
    (.updatedAt | fromdateiso8601) < (now - 604800)
  ) | "\(.number) \(.title) (updated: \(.updatedAt[:10]))"'
```

**GitLab:**
```bash
glab mr list -F json | jq '[.[] | select((.updated_at | fromdateiso8601) < (now - 604800)) | {iid:.iid,title:.title,updated:.updated_at}]'
```
