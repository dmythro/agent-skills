# PR/MR Review Comment Workflow

Opinionated workflow for handling review comments on both GitHub PRs and GitLab MRs. Follow this when asked to "check PR comments", "address review feedback", "handle MR comments", or "resolve review threads".

**The workflow has two strict phases -- never mix them:**

1. **Phase 1: Analyze and Fix** -- fetch threads, research each comment, make all code fixes, commit and push. No GitHub API writes in this phase.
2. **Phase 2: Reply and Resolve** -- reply to each thread and resolve handled ones, all in a single batched command.

This ordering matters: pushing fixes first ensures reviewers see the changes when they read replies. Never reply claiming "Fixed" before the fix is actually pushed.

**Speed optimization**: Phase 1 fetching requires zero manual approvals (all commands are allowlisted read-only). Phase 2 combines all replies and all resolves into one command -- one approval for the entire phase.

## 1. Fetch Unresolved Review Threads

Always start by fetching **unresolved** threads. Don't show already-resolved threads unless explicitly asked.

### GitHub: Single GraphQL Fetch (Zero Approvals)

One command, zero approvals. This single call returns all threads, comments, and resolution status -- never split into separate count and data calls.

**Generate as a single line.** Allowlist `*` patterns may not match across newlines (undocumented behavior). GraphQL ignores whitespace, so a one-liner works fine. Use `$(...)` substitution for owner, repo, and PR number inline.

```bash
gh api graphql -f query="{ repository(owner: \"$(gh repo view --json owner --jq '.owner.login')\", name: \"$(gh repo view --json name --jq '.name')\") { pullRequest(number: $(gh pr view --json number --jq '.number')) { reviewThreads(first: 100) { totalCount pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated path line startLine diffSide comments(first: 20) { nodes { id fullDatabaseId body author { login } createdAt outdated replyTo { id } } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads | {total: .totalCount, nextCursor: (if .pageInfo.hasNextPage then .pageInfo.endCursor else null end), unresolved: [.nodes[] | select(.isResolved==false)]}'
```

Returns the unresolved threads, plus the evidence that the fetch was complete. **`reviewThreads` does not paginate on its own**: a non-null `nextCursor` means threads 101+ were never fetched and may hold unresolved comments -- re-run with `reviewThreads(first: 100, after: \"{nextCursor}\")` and merge. `comments(first: 20)` truncates the same way on very long threads.

**Critical rules:**

- **Generate as a single line** -- `*` may not match across newlines
- Do NOT assign shell variables (`OWNER=...`, `REPO=...`) before `gh api graphql` -- breaks pattern matching
- Do NOT use GraphQL `$` variables (`$owner`, `$repo`) in double-quoted query strings -- conflicts with shell `$` expansion
- Do NOT pipe to `jq` -- use the `--jq` flag instead (pipes break allowlist matching)

**Response field mapping (critical -- using wrong ID causes silent failures):**

| Field              | Format                  | Use for                      |
|--------------------|-------------------------|------------------------------|
| thread `.id`       | `PRRT_...` (node ID)    | `resolveReviewThread` mutation |
| comment `.fullDatabaseId` | `"2949637341"` (string) | REST reply endpoint          |
| comment `.id`      | `PRRC_...` (node ID)    | Not typically needed          |

### GitLab: Single REST Call

```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

### Bot Findings That Are Not Threads (GitHub)

**Threads are not the whole review.** CodeRabbit posts only its *actionable* findings inline; nitpicks (under `profile: chill`), findings outside the diff range, duplicates of earlier findings, and comments that failed to post are written into the **review body**, where the query above cannot see them. Skipping them is the usual reason a review looks handled and the reviewer has to ask again. Add one call per round:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --paginate --slurp | jq -r --arg p coderabbit '.[][] | select((.user.login|ascii_downcase|startswith($p)) and ((.body|length)>0)) | (.body|ascii_downcase) as $b | "\(.submitted_at) \(.commit_id[0:7]) inline=\(($b|capture("actionable comments posted:[ *]*(?<n>[0-9]+)")? // {n:"?"}).n) body-only=[\([$b|scan("(nitpick comments|outside diff range comments|duplicate comments|comments failed to post) *\\(([0-9]+)\\)")|join(" ")]|join("; "))]"'
```

Read the full body of any review with a non-zero bucket and triage each item like a thread. They cannot be resolved (no thread exists), so state the verdicts in a reply on a related thread or in one PR comment. The inline set itself is complete only when the top-level bot threads equal the sum of `Actionable comments posted: N`; see `bot-review-loop.md` for the reconciliation command.

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

1. Fix all "valid and unaddressed" comments in code -- threads and body-only findings alike. Do not add code comments that narrate the fix or restate what the code already reads -- comment only non-obvious constraints.
2. Commit the fixes using Conventional Commits format, as many commits as the change needs. The message describes the change itself (e.g., `fix: handle null token refresh`), never the review process -- no "address review feedback", bot names, or round numbers; the PR threads already record why (see the `git-commit` skill).
3. **Push once, after the last commit of the round -- and only when the round is genuinely finished.** On a repo with auto-review every push triggers another incremental bot review from a per-developer hourly bucket, and it reviews whatever is on the branch at that moment. So the push waits for *everything*: every valid finding fixed (threads and body-only), every rejected one with a verdict, the tests and docs the fixes imply, local checks green, nothing left uncommitted. Pushing per commit -- or pushing to check one fix -- spends several reviews on unfinished work. If the change is still moving, keep the PR a draft -- drafts are not auto-reviewed, so pushes to them are free. Full gate: `bot-review-loop.md` (One Push Per Round).

Only proceed to Phase 2 after the push succeeds. This ensures reviewers see the actual fixes when they read your replies.

## 4. Reply and Resolve All Threads (Phase 2 -- One Approval)

**Combine all replies and all resolves into a single command.** Chain REST replies with `&&`, then append a single GraphQL mutation that batch-resolves all handled threads using aliases. The user approves once for the entire phase.

### GitHub: Batched Replies + Resolves

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_1}/replies -f body="Fixed in {sha} -- {explanation}" && \
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_2}/replies -f body="Addressed in {sha} -- {description}" && \
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_3}/replies -f body="Intentional -- {evidence}" && \
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_4}/replies -f body="Good point. {analysis}. Leaving for your call." && \
gh api graphql -f query="
mutation {
  t1: resolveReviewThread(input: {threadId: \"PRRT_thread1\"}) {
    thread { isResolved }
  }
  t2: resolveReviewThread(input: {threadId: \"PRRT_thread2\"}) {
    thread { isResolved }
  }
  t3: resolveReviewThread(input: {threadId: \"PRRT_thread3\"}) {
    thread { isResolved }
  }
}"
```

**Rules:**

- Use comment `fullDatabaseId` for reply endpoints, NOT the node `id` (`databaseId` is deprecated -- it cannot hold 64-bit ids; `fullDatabaseId` returns the same digits as a string)
- Use thread `.id` (starts with `PRRT_`) for resolveReviewThread, NOT the comment ID
- Add one aliased operation (`t1`, `t2`, ...) per thread to resolve
- Do NOT include "Needs discussion" threads in the resolve mutation -- leave those for the reviewer
- All replies come before the resolve mutation so reviewers see context before resolution
- Keep replies to one or two sentences: verdict + evidence. No filler

### Reply Content by Classification

| Classification     | Reply template                                                     | Resolve? |
|--------------------|--------------------------------------------------------------------|----------|
| Valid + Fixed      | `Fixed in {sha} -- {brief explanation}.`                           | Yes      |
| Already addressed  | `Addressed in {sha} -- {brief description}.`                      | Yes      |
| Incorrect/Outdated | `Intentional -- {reference to docs/convention}. {explanation}.`    | Yes      |
| Needs discussion   | `Good point. {analysis}. Leaving for your call.`                   | No       |

### GitLab: Batched Replies + Resolves

```bash
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1}/notes --method POST --field "body=Fixed in {sha}" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2}/notes --method POST --field "body=Addressed in {sha}" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_3}/notes --method POST --field "body=Intentional -- {evidence}" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1} --method PUT --field "resolved=true" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2} --method PUT --field "resolved=true" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_3} --method PUT --field "resolved=true"
```

### Unresolve a Thread (If Needed)

**GitHub:**
```bash
gh api graphql -f query="
mutation {
  unresolveReviewThread(input: {threadId: \"PRRT_...\"}) {
    thread { isResolved }
  }
}"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions/{discussion_id} \
  --method PUT --field "resolved=false"
```

## 5. Summary Checklist

After both phases are complete:

- All comments evaluated with full context (file, surrounding code, git log, conventions)
- Body-only bot findings (nitpicks, outside diff range, failed to post) triaged too, with verdicts stated somewhere on the PR
- Top-level bot threads reconcile with the `Actionable comments posted: N` the reviews claim
- All valid feedback addressed with code fixes
- Fixes committed and pushed before any replies, in a single push for the whole round, made only once every task of the round was done (pre-push gate in `bot-review-loop.md`)
- All replies and resolves batched into one command (one approval)
- "Needs discussion" threads replied to but left unresolved
- No reply says "Fixed" without a corresponding pushed commit
- Unresolved threads re-fetched after the final push -- the push triggers another incremental review, which lands minutes later and is the round you are most likely to abandon half-read
- That follow-up round confirmed as having actually run -- a green `CodeRabbit` check whose `description` reads `Review rate limited` means it never did (`bot-review-loop.md`, The Status Check Nobody Reads)
