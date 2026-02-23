# Read-Only gh api Endpoints

Reference for `gh api` calls that are safe GET requests — read-only, no side effects.

## Key Guidance

- **Always prefer subcommands** (`gh pr view`, `gh issue list`, etc.) over `gh api` when possible. Subcommands are auto-approvable and handle pagination/errors.
- **Only use `gh api`** when subcommands can't do it (line comments, thread resolution, GraphQL)
- **`gh api` defaults to GET.** Write calls always need explicit `--method POST/PUT/PATCH/DELETE`.

## Pull Request Endpoints (GET)

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

# Merge status
gh api repos/{owner}/{repo}/pulls/{pr}/merge
# Returns 204 if merged, 404 if not
```

## Issue Endpoints (GET)

```bash
# Issue details
gh api repos/{owner}/{repo}/issues/{issue}

# Issue comments
gh api repos/{owner}/{repo}/issues/{issue}/comments
gh api repos/{owner}/{repo}/issues/{issue}/comments --paginate

# Issue labels
gh api repos/{owner}/{repo}/issues/{issue}/labels

# Issue timeline (events)
gh api repos/{owner}/{repo}/issues/{issue}/timeline --paginate

# Issue reactions
gh api repos/{owner}/{repo}/issues/{issue}/reactions
```

## Actions Endpoints (GET)

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

## GraphQL Read-Only Queries

GraphQL uses POST method but these are read-only queries:

```bash
# Review threads with resolution status
gh api graphql -f query='
{
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: 123) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          comments(first: 10) {
            nodes {
              id
              databaseId
              body
              author { login }
              createdAt
            }
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

## Pagination

For REST endpoints returning lists, use `--paginate` to get all results:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate
```

For GraphQL, use cursor-based pagination:

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
