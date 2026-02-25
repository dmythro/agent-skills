# Read-Only API Endpoints

Reference for read-only API calls across GitHub (`gh api`) and GitLab (`glab api`). Use these only when CLI subcommands cannot provide the data.

## Key Guidance

- **Always prefer subcommands** (`gh pr view`, `glab mr view`, etc.) over raw API calls. Subcommands are auto-approvable and handle pagination/errors.
- **Only use API calls** when subcommands can't do it (line comments, thread resolution, GraphQL)
- **`gh api` defaults to GET.** Write calls need explicit `--method POST/PUT/PATCH/DELETE`.
- **`glab api` defaults to GET.** Same convention for write operations.

## GitHub Pull Request Endpoints (GET)

```bash
# PR details
gh api repos/{owner}/{repo}/pulls/{pr}

# PR comments (review comments, on specific lines)
gh api repos/{owner}/{repo}/pulls/{pr}/comments
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate

# PR reviews
gh api repos/{owner}/{repo}/pulls/{pr}/reviews
gh api repos/{owner}/{repo}/pulls/{pr}/reviews/{review_id}/comments

# PR files changed
gh api repos/{owner}/{repo}/pulls/{pr}/files
gh api repos/{owner}/{repo}/pulls/{pr}/files --paginate

# PR commits
gh api repos/{owner}/{repo}/pulls/{pr}/commits

# Requested reviewers
gh api repos/{owner}/{repo}/pulls/{pr}/requested_reviewers

# Merge status (204 if merged, 404 if not)
gh api repos/{owner}/{repo}/pulls/{pr}/merge
```

## GitHub Issue Endpoints (GET)

```bash
# Issue details
gh api repos/{owner}/{repo}/issues/{issue}

# Issue comments
gh api repos/{owner}/{repo}/issues/{issue}/comments --paginate

# Issue labels
gh api repos/{owner}/{repo}/issues/{issue}/labels

# Issue timeline (events)
gh api repos/{owner}/{repo}/issues/{issue}/timeline --paginate

# Issue reactions
gh api repos/{owner}/{repo}/issues/{issue}/reactions
```

## GitHub Actions Endpoints (GET)

```bash
# Workflow run details
gh api repos/{owner}/{repo}/actions/runs/{run_id}

# Workflow run jobs
gh api repos/{owner}/{repo}/actions/runs/{run_id}/jobs

# Workflow run logs (returns redirect to download URL)
gh api repos/{owner}/{repo}/actions/runs/{run_id}/logs

# List workflow runs
gh api repos/{owner}/{repo}/actions/runs

# Workflow run artifacts
gh api repos/{owner}/{repo}/actions/runs/{run_id}/artifacts
```

## GitHub GraphQL Read-Only Queries

GraphQL uses POST method but these are read-only queries:

```bash
# Review threads with resolution status
gh api graphql -f query='
{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: 123) {
      reviewThreads(first: 100) {
        nodes {
          id, isResolved, isOutdated, path, line
          comments(first: 10) {
            nodes { id, databaseId, body, author { login }, createdAt }
          }
        }
      }
    }
  }
}'

# PR review decision and merge state
gh api graphql -f query='
{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: 123) {
      reviewDecision
      mergeable
      mergeStateStatus
      isDraft
    }
  }
}'
```

## GitLab Merge Request Endpoints (GET)

```bash
# MR details
glab api projects/{project_id}/merge_requests/{iid}

# MR changes (files diff)
glab api projects/{project_id}/merge_requests/{iid}/changes

# MR commits
glab api projects/{project_id}/merge_requests/{iid}/commits

# MR discussions (review threads)
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate

# MR notes (comments)
glab api projects/{project_id}/merge_requests/{iid}/notes --paginate

# MR approvals
glab api projects/{project_id}/merge_requests/{iid}/approvals

# MR participants
glab api projects/{project_id}/merge_requests/{iid}/participants

# MR pipelines
glab api projects/{project_id}/merge_requests/{iid}/pipelines
```

## GitLab Issue Endpoints (GET)

```bash
# Issue details
glab api projects/{project_id}/issues/{iid}

# Issue notes (comments)
glab api projects/{project_id}/issues/{iid}/notes --paginate

# Issue labels
glab api projects/{project_id}/issues/{iid}/resource_label_events
```

## Pagination

### GitHub REST

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate
```

### GitHub GraphQL

```bash
gh api graphql -f query='
{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: 123) {
      comments(first: 100, after: "CURSOR") {
        pageInfo { hasNextPage endCursor }
        nodes { body author { login } }
      }
    }
  }
}'
```

### GitLab REST

```bash
# API endpoint pagination
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate

# Subcommand pagination
glab mr list -P 100
```
