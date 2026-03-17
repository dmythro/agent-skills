# PR/MR Review Comment Workflow

Opinionated workflow for handling review comments on both GitHub PRs and GitLab MRs. Follow this when asked to "check PR comments", "address review feedback", "handle MR comments", or "resolve review threads".

**The workflow has two strict phases -- never mix them:**

1. **Phase 1: Analyze and Fix** -- fetch threads, research each comment, make all code fixes, commit and push. No GitHub API writes in this phase.
2. **Phase 2: Reply and Resolve** -- reply to each thread, resolve handled ones. Only after fixes are pushed.

This ordering matters: pushing fixes first ensures reviewers see the changes when they read replies. Never reply claiming "Fixed" before the fix is actually pushed.

## 1. Fetch Unresolved Review Threads

Always start by fetching **unresolved** threads. Don't show already-resolved threads unless explicitly asked.

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

### GitHub (GraphQL) -- shell variables, no GraphQL `$` variables

Do NOT use GraphQL variables (`$owner`, `$repo`, `$pr`) in these double-quoted query strings -- they conflict with shell variable expansion (`$OWNER`, `$REPO`, `$PR`). The shell expands all `$` references before `gh` receives the query, so GraphQL `$` declarations would be consumed by the shell. Instead, inline values directly via shell variables:

```bash
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
PR={number}

gh api graphql -f query="
{
  repository(owner: \"$OWNER\", name: \"$REPO\") {
    pullRequest(number: $PR) {
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
}"
```

Filter to unresolved threads:
```bash
... | jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)]'
```

**Response field mapping (critical -- using wrong ID causes silent failures):**

| Field              | Format                  | Use for                      |
|--------------------|-------------------------|------------------------------|
| thread `.id`       | `PRRT_...` (node ID)    | `resolveReviewThread` mutation |
| comment `.databaseId` | `2949637341` (numeric)  | REST reply endpoint          |
| comment `.id`      | `PRRC_...` (node ID)    | Not typically needed          |

### GitLab (REST)

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

## 2. Evaluate Each Comment (Phase 1)

For each unresolved thread, build full context before making any judgment:

### Research Checklist (do all of these before deciding)

1. **Read the file** at the exact path and line the comment references
2. **Read surrounding context** -- at least 20 lines above and below, more if the comment is about a pattern or architecture
3. **Check current state vs comment time** -- the code may have changed since the comment was posted. Use `git log --oneline {path}` to see if relevant commits landed after the review
4. **Check project conventions** -- read CLAUDE.md, AGENTS.md, contributing guides, and look at how similar code is written elsewhere in the codebase
5. **Verify factual claims** -- if the reviewer claims an API works differently, a pattern is wrong, or a flag doesn't exist, verify it. Reviewers can be wrong about details.
6. **Check if this is an automated review** -- bot reviewers (Copilot, CodeRabbit, etc.) are useful but frequently produce false positives. Be extra critical of automated comments.

### Classify the Comment

Only after completing the research checklist, classify:

- **Valid and unaddressed** -- the reviewer is right and the code needs a fix
- **Already addressed** -- a subsequent commit already fixed this
- **Incorrect** -- the reviewer's claim is factually wrong (you must have evidence)
- **Style preference** -- valid opinion but not blocking, not a correctness issue
- **Needs discussion** -- legitimate disagreement where the human should decide

The goal is 100% confidence in your verdict before acting. If you're unsure, classify as "Needs discussion" rather than guessing.

## 3. Fix All Issues (Phase 1 continued)

After evaluating all comments, make all code fixes before any GitHub API writes:

1. Fix all "valid and unaddressed" comments in code
2. Commit the fixes using Conventional Commits format (e.g., `fix: address PR review feedback`)
3. Push the commit

Only proceed to Phase 2 after the push succeeds. This ensures reviewers see the actual fixes when they read your replies.

## 4. Reply and Resolve All Threads (Phase 2)

Now reply to every thread and resolve the ones that are handled. Do this in a batch after pushing.

### Valid + Fixed: Reply, Resolve

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_databaseId}/replies \
  -f body="Fixed in {commit_sha} -- [brief explanation of what changed]."
```

Then resolve (see Key Commands Reference below).

### Already Addressed: Reply, Resolve

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_databaseId}/replies \
  -f body="This was addressed in {commit_sha} -- [brief description]."
```

Then resolve.

### Invalid/Outdated: Reply with Evidence, Resolve

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_databaseId}/replies \
  -f body="This is actually intentional -- [reference to convention/docs/code pattern]. [Explanation]."
```

Then resolve. Provide specific evidence: file paths, documentation references, or code examples.

### Needs Discussion: Reply, Leave Unresolved

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_databaseId}/replies \
  -f body="Good point. [Your analysis]. I think [recommendation], but leaving this for your call."
```

**Do not resolve** -- leave for human decision.

### GitLab Equivalents

Reply:
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body={reply text}"
```

Resolve:
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
```

## 5. Key Commands Reference

### Reply to a Comment Thread

The reply endpoint uses `databaseId` (numeric) from the GraphQL response, NOT the node `id`. Use the `databaseId` of the first comment in the thread (or any comment you're replying to):

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_databaseId}/replies \
  -f body="Reply text"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id}/notes \
  --method POST --field "body=Reply text"
```

### Resolve a Thread

**GitHub** -- use the thread `id` (node ID, starts with `PRRT_`), NOT the comment ID:
```bash
THREAD_ID="PRRT_..."
gh api graphql -f query="
mutation {
  resolveReviewThread(input: { threadId: \"$THREAD_ID\" }) {
    thread { isResolved }
  }
}"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=true"
```

### Unresolve a Thread (If Needed)

**GitHub:**
```bash
THREAD_ID="PRRT_..."
gh api graphql -f query="
mutation {
  unresolveReviewThread(input: { threadId: \"$THREAD_ID\" }) {
    thread { isResolved }
  }
}"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=false"
```

## 6. Summary Checklist

After both phases are complete:

- All comments evaluated with full context (file, surrounding code, git log, conventions)
- All valid feedback addressed with code fixes
- Fixes committed and pushed before any replies
- All threads replied to with concise explanations referencing specific commits
- Handled threads resolved, discussion threads left unresolved
- No reply says "Fixed" without a corresponding pushed commit
