# PR/MR Review Comment Workflow

Opinionated workflow for handling review comments on both GitHub PRs and GitLab MRs. Follow this when asked to "check PR comments", "address review feedback", "handle MR comments", or "resolve review threads".

## 1. Fetch Unresolved Review Threads

Always start by fetching **unresolved** threads. Don't show already-resolved threads unless explicitly asked.

### GitHub (GraphQL)

```bash
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

### GitLab (REST)

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

### Get Owner/Repo Automatically

**GitHub:**
```bash
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
```

**GitLab:**
```bash
# Project ID from current repo
glab api projects/:fullpath | jq '.id'
```

## 2. Evaluate Each Comment

For each unresolved thread:

### Read the Code Context

- Open the file at the path referenced by the comment
- Read the specific lines the comment refers to (use `line` and `startLine`)
- Check the current state of the code on the PR/MR branch

### Check if Already Addressed

Compare the comment's concerns against the current code:
- Has the code been changed since the comment was posted?
- Does a subsequent commit already address the feedback?
- Check with `gh pr diff` / `glab mr diff` or read the file directly

### Validate the Comment

Be critical -- don't blindly agree with every comment. Evaluate against:
- **Actual codebase patterns** -- does the project already do it this way?
- **Project conventions** -- check CLAUDE.md, style guides, existing code
- **Correctness** -- is the reviewer's assumption actually correct?
- **Priority** -- is this a real issue or a style preference?

Comments may be:
- **Valid and unaddressed** -- needs a fix
- **Already addressed** -- code changed since comment was posted
- **Incorrect** -- reviewer's assumption is wrong or outdated
- **Style preference** -- valid but low-priority, not blocking
- **Needs discussion** -- legitimate disagreement requiring human input

## 3. Act on Each Comment

### Valid + Not Addressed: Fix, Reply, Resolve

1. Fix the code
2. Reply to the thread explaining the fix
3. Resolve the thread

**GitHub:**
```bash
# Reply to the review thread
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Fixed -- [brief explanation of what changed]."

# Resolve the thread (GraphQL)
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="PRRT_..."
```

**GitLab:**
```bash
# Reply to the discussion
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Fixed -- [brief explanation of what changed]."

# Resolve the discussion
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
```

### Already Addressed: Reply, Resolve

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="This was addressed in [commit ref] -- [brief description]."
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=This was addressed in [commit ref] -- [brief description]."
```

Then resolve the thread/discussion.

### Invalid/Outdated: Reply with Evidence, Resolve

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="This is actually intentional -- [reference to convention/docs/code pattern]. [Explanation]."
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=This is actually intentional -- [reference to convention/docs/code pattern]. [Explanation]."
```

Then resolve. Provide specific evidence: file paths, documentation references, or code examples.

### Needs Discussion: Reply, Leave Unresolved

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Good point. [Your analysis]. I think [recommendation], but leaving this for your call."
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Good point. [Your analysis]. I think [recommendation], but leaving this for your call."
```

**Do not resolve** -- leave for human decision.

## 4. Key Commands Reference

### Reply to a Comment Thread

The `comment_id` in the GitHub REST API is the `databaseId` from the GraphQL response (the first comment in the thread, or any comment you're replying to):

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies \
  -f body="Reply text"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Reply text"
```

### Resolve a Thread

**GitHub** -- use the GraphQL thread `id` (node ID, starts with `PRRT_`):
```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="PRRT_..."
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
```

### Unresolve a Thread (If Needed)

**GitHub:**
```bash
gh api graphql -f query='
mutation($threadId: ID!) {
  unresolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId="PRRT_..."
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=false"
```

## 5. Summary Checklist

After processing all comments:

- All valid feedback addressed with code fixes
- All threads replied to with clear explanations
- Resolved threads that are definitively handled
- Left unresolved only threads needing human decision
- Committed fixes using Conventional Commits format (if any code changes were made)
