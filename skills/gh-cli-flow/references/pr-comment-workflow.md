# PR Review Comment Workflow

Opinionated workflow for handling PR review comments. Follow this when asked to "check PR comments", "address review feedback", or "handle PR comments".

## 1. Fetch Unresolved Review Threads

Always start by fetching **unresolved** threads. Don't show already-resolved threads unless explicitly asked.

```bash
# Fetch unresolved review threads with full context
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          diffSide
          comments(first: 20) {
            nodes {
              id
              databaseId
              body
              author { login }
              createdAt
              outdated
              replyTo { id }
            }
          }
        }
      }
    }
  }
}' -f owner=OWNER -f repo=REPO -F pr=123
```

Filter to `isResolved: false` threads from the response.

## 2. Evaluate Each Comment

For each unresolved thread:

### a. Read the Code Context

- Open the file at the path referenced by the comment
- Read the specific lines the comment refers to (use `line` and `startLine`)
- Check the current state of the code on the PR branch

### b. Check if Already Addressed

Compare the comment's concerns against the current code:
- Has the code been changed since the comment was posted?
- Does a subsequent commit already address the feedback?
- Check with `gh pr diff` or read the file directly

### c. Validate the Comment

Be critical — don't blindly agree with every comment. Evaluate against:
- **Actual codebase patterns** — does the project already do it this way?
- **Project conventions** — check CLAUDE.md, style guides, existing code
- **Correctness** — is the reviewer's assumption actually correct?
- **Priority** — is this a real issue or a style preference?

Comments may be:
- **Valid and unaddressed** — needs a fix
- **Already addressed** — code changed since comment was posted
- **Incorrect** — reviewer's assumption is wrong or outdated
- **Style preference** — valid but low-priority, not blocking
- **Needs discussion** — legitimate disagreement requiring human input

## 3. Act on Each Comment

### Valid + Not Addressed → Fix, Reply, Resolve

1. Fix the code
2. Reply to the thread explaining the fix
3. Resolve the thread

```bash
# Reply to the review thread
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Fixed — [brief explanation of what changed]."

# Resolve the thread (GraphQL)
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="THREAD_NODE_ID"
```

### Already Addressed → Reply, Resolve

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="This was addressed in [commit ref] — [brief description]."
```

Then resolve the thread.

### Invalid/Outdated → Reply with Evidence, Resolve

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="This is actually intentional — [reference to convention/docs/code pattern]. [Explanation]."
```

Then resolve the thread. Provide specific evidence: file paths, documentation references, or code examples.

### Needs Discussion → Reply, Leave Unresolved

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Good point. [Your analysis]. I think [recommendation], but leaving this for your call."
```

**Do not resolve** — leave for human decision.

## 4. Key Commands Reference

### Get PR Owner/Repo Automatically

`gh` auto-detects the repo from git remote. For API calls, extract owner/repo:

```bash
# Get owner and repo from current directory
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
```

### Reply to a Comment Thread

The `comment_id` in the REST API is the `databaseId` from the GraphQL response (the first comment in the thread, or any comment you're replying to):

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Reply text"
```

### Resolve a Thread

Use the GraphQL thread `id` (node ID, starts with `PRRT_`):

```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="PRRT_..."
```

### Unresolve a Thread (if needed)

```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  unresolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="PRRT_..."
```

## 5. Summary Checklist

After processing all comments:

- [ ] All valid feedback addressed with code fixes
- [ ] All threads replied to with clear explanations
- [ ] Resolved threads that are definitively handled
- [ ] Left unresolved only threads needing human decision
- [ ] Committed fixes using Conventional Commits format (if any code changes were made)
