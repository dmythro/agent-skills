# Review State & CI Queries

Advanced patterns for querying PR review state, CI status, and merge readiness.

## Review State Summary

### Latest Review Per Reviewer

Each reviewer can submit multiple reviews. Get the most recent state per reviewer:

```bash
gh pr view 123 --json reviews --jq '
  .reviews
  | group_by(.author.login)
  | map({
      user: .[0].author.login,
      latest: .[-1].state,
      count: length
    })
  | sort_by(.user)'
```

States: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, `PENDING`, `DISMISSED`.

### Quick Approval Check

```bash
# Is the PR approved by at least one reviewer?
gh pr view 123 --json reviews --jq '
  [.reviews[] | select(.state == "APPROVED")] | length > 0'

# Who approved?
gh pr view 123 --json reviews --jq '
  [.reviews[] | select(.state == "APPROVED") | .author.login] | unique'

# Who requested changes?
gh pr view 123 --json reviews --jq '
  [.reviews[] | select(.state == "CHANGES_REQUESTED") | .author.login] | unique'
```

### Pending Reviewers

```bash
# Requested but haven't reviewed yet
gh pr view 123 --json reviewRequests --jq '.reviewRequests[].login'

# Or with type info (user vs team)
gh pr view 123 --json reviewRequests --jq '.reviewRequests[] | {login, type: .type}'
```

## CI / Check Status

### All Checks Summary

```bash
gh pr checks 123 --json name,state,conclusion,startedAt,completedAt
```

### Failing Checks

```bash
gh pr checks 123 --json name,conclusion --jq '
  .[] | select(.conclusion == "FAILURE") | .name'
```

### Required Checks Status

```bash
# Check if all required status checks pass
gh pr view 123 --json statusCheckRollup --jq '
  .statusCheckRollup[] | {name: .name, status: .status, conclusion: .conclusion}'
```

### Watch Until Complete

```bash
# Block until all checks finish
gh pr checks 123 --watch --interval 10

# Watch with fail-fast
gh pr checks 123 --watch --fail-fast
```

## Merge Readiness

### Full Readiness Assessment

```bash
# Mergeable state + review decision + checks
gh pr view 123 --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus
```

Fields:
- `mergeable` — `MERGEABLE`, `CONFLICTING`, or `UNKNOWN`
- `reviewDecision` — `APPROVED`, `CHANGES_REQUESTED`, `REVIEW_REQUIRED`, or empty
- `isDraft` — boolean
- `mergeStateStatus` — `CLEAN`, `BLOCKED`, `BEHIND`, `DIRTY`, `UNSTABLE`

### Conflict Detection

```bash
gh pr view 123 --json mergeable --jq '.mergeable'
# MERGEABLE | CONFLICTING | UNKNOWN
```

## PR Metadata

### Files Changed

```bash
# File list with stats
gh pr view 123 --json files --jq '
  .files[] | "\(.additions)+/\(.deletions)- \(.path)"'

# Just filenames
gh pr diff 123 --name-only

# Diffstat
gh pr diff 123 --stat

# Count of changed files
gh pr view 123 --json files --jq '.files | length'
```

### Commit History

```bash
# PR commits
gh pr view 123 --json commits --jq '
  .commits[] | "\(.oid[:7]) \(.messageHeadline)"'

# Commit count
gh pr view 123 --json commits --jq '.commits | length'
```

### Labels and Milestones

```bash
# Labels
gh pr view 123 --json labels --jq '[.labels[].name] | join(", ")'

# Milestone
gh pr view 123 --json milestone --jq '.milestone.title'
```

### Linked Issues

```bash
# Issues that this PR closes (from body "Closes #123" etc.)
gh pr view 123 --json closingIssuesReferences --jq '
  .closingIssuesReferences[] | "#\(.number) \(.title)"'
```

### PR Age and Activity

```bash
# Creation date, last update, author
gh pr view 123 --json createdAt,updatedAt,author --jq '
  "Created: \(.createdAt)\nUpdated: \(.updatedAt)\nAuthor: \(.author.login)"'
```

## Batch Queries

### All Open PRs Needing Review

```bash
gh pr list --search "review-requested:@me" --json number,title,url
```

### PRs with Failing CI

```bash
gh pr list --json number,title,statusCheckRollup --jq '
  .[] | select(.statusCheckRollup | any(.conclusion == "FAILURE"))
  | "\(.number) \(.title)"'
```

### Stale PRs (No Activity in 7+ Days)

```bash
gh pr list --json number,title,updatedAt --jq '
  .[] | select(
    (.updatedAt | fromdateiso8601) < (now - 604800)
  ) | "\(.number) \(.title) (updated: \(.updatedAt[:10]))"'
```
