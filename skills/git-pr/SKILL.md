---
name: git-pr
description: >-
  PR and MR workflows for GitHub (gh) and GitLab (glab). Creation, review
  comment handling, thread resolution, review state queries, merging,
  cost-aware bot review rounds (GitHub: Copilot, CodeRabbit), and Copilot
  code review configuration (rulesets, custom instructions, billing). Use
  when creating PRs/MRs, addressing review feedback, resolving threads,
  looping bot reviews, checking approvals, querying PR data, configuring
  Copilot reviews, or configuring gh/glab read-only allowlists. Not for
  git commits (git-commit), CI/CD status (git-ci), local pre-push
  CodeRabbit CLI reviews (coderabbit), or general git ops
---

# PR and MR Workflows

**Primary skill for pull request and merge request workflows across GitHub and GitLab.** Covers the full lifecycle: creation, review queries, comment handling, line-specific comments, and merging. All recipes use minimal field sets for token efficiency.

## Context Check (Do This First)

Before starting any PR workflow, detect the current state. This determines the right action:

```bash
# Check if a PR exists for the current branch (allowlisted, zero approval)
gh pr view --json number,state,reviewDecision,reviewRequests,title 2>/dev/null
```

| Result                                     | Next action                                                |
|--------------------------------------------|------------------------------------------------------------|
| PR exists, `CHANGES_REQUESTED`             | Fetch unresolved threads (see Review Comment Handling)      |
| PR exists, `REVIEW_REQUIRED` or has pending `reviewRequests` | Check review state or wait for reviewers  |
| PR exists, a bot pending in REST `requested_reviewers` (Copilot shows as `Copilot`) | Auto-review in flight -- wait for it (Bot Review Loop); do NOT re-request |
| PR exists, unresolved review comments      | Address the comments first (Review Comment Handling); never request a new review over outstanding ones |
| PR exists, `APPROVED`                      | Check CI status or proceed with merge                      |
| PR exists, no review decision yet          | Check CI status, review state, or push more changes        |
| No PR for current branch                   | Create a PR (see PR/MR Creation)                           |

This avoids offering to create a PR when one already exists, and immediately surfaces pending review work. The `reviewDecision` field reliably indicates whether a reviewer has requested changes without needing to fetch individual threads.

**Auto-review is an optional setting -- never assume it either way (GitHub):** automatic Copilot code review comes from either a repo/org **ruleset** or the author's **personal** Copilot setting, so its presence varies between accounts, orgs, and even repos of the same owner. When enabled it requests Copilot on every **non-draft** PR at creation (and when a draft is marked ready for review), so a freshly created PR may already have a pending bot request or bot comments minutes later -- and on other repos nothing fires at all. The ruleset source is detectable read-only (the `copilot_code_review` rule via `gh api repos/{owner}/{repo}/rules/branches/{branch}`; see `references/copilot-review-config.md`), but the personal setting has no API -- so an empty ruleset check still means **detect, don't assume**: check for a pending request or an existing bot review first, and only request one if neither shows up; otherwise you get a duplicate round. **Gotcha:** the pending bot request is only visible via `gh api repos/{owner}/{repo}/pulls/{n}/requested_reviewers` (login `Copilot`) -- `gh pr view --json reviewRequests` omits bot reviewers and stays empty.

**Bot reviews cost money (Copilot) or quota (CodeRabbit) -- budget rounds deliberately.** Every Copilot review, including each re-request, bills fully: 13 premium requests on legacy annual plans (up to ~23 reviews/month on Pro -- the pool is shared with all premium features), or token-metered AI credits + Actions minutes on current plans. CodeRabbit PR reviews draw from an hourly per-plan bucket that is **per developer, not per PR** -- all of a developer's open PRs contend for the same review windows. Check the project's **Code Review Policy** (below) before requesting anything billable; prefer local CodeRabbit CLI reviews (see the `coderabbit` skill) to iterate cheaply before the PR-side review.

## When to Use

- **Creating PRs or MRs** -- draft workflows, fill patterns, title format
- **Addressing PR/MR review feedback** -- "fix PR comments", "address review", "handle feedback", "resolve review threads"
- **Responding to code review** -- evaluating reviewer comments, replying, resolving threads
- **Checking review state** -- approvals, pending reviewers, review decisions
- **Querying PR/MR data** -- files changed, commits, labels, linked issues
- **Posting comments on PRs/MRs** -- line-specific comments, thread replies
- **Looping bot review rounds** (GitHub only; Copilot, CodeRabbit) -- re-request the bot, wait for the async review, address comments, repeat until no valid comments remain ("loop the Copilot review", "loop 3 rounds")
- **Configuring Copilot code review** -- auto-review rulesets, review effort, custom instructions, billing/quota checks
- **Configuring tool allowlists** -- auto-approval patterns for read-only commands

## Critical Rules

1. **Prefer CLI subcommands over raw API calls.** Subcommands handle pagination, error formatting, and repo detection. Only use `gh api` / `glab api` for operations not covered by subcommands (line comments, thread resolution, GraphQL).
2. **Use `--json field1,field2` with `gh` to filter output.** This IS the efficiency mechanism -- no `--jq` needed for basic queries. Only request fields you actually need.
3. **`glab` has no `--json field1,field2` equivalent.** Use `-F json | jq '{fields}'` to filter output for token efficiency.
4. **Use commands exactly as shown in this skill.** The commands below are designed to match auto-approval allowlist patterns. Improvising flag order or adding unexpected flags may trigger permission prompts.
5. **Push only finished work -- one push per round.** Where auto-review and `auto_incremental_review` are on (both default), `git push` **is** the review request and it reviews whatever is on the branch at that moment. Do every task of the round first -- all fixes, tests, docs, local checks (`coderabbit review --committed`), everything committed -- then push once. A progress push spends a scarce per-developer review window on code you already intend to change. While the change is still moving, keep the PR a draft (drafts are not auto-reviewed, so pushes to them are free); marking it ready is the review request. Full gate: `references/bot-review-loop.md` (One Push Per Round).
6. **A review is not handled until its body-only findings are.** Bot findings that never become threads (CodeRabbit nitpick, outside-diff-range, duplicate and failed-to-post buckets) live in the review body and are invisible to thread queries -- triage them too, and reconcile the claimed comment count before declaring a round done.
7. **A green bot check is not a completed review.** CodeRabbit records each round as a commit status that is `success` either way -- the outcome is only in its `description` (`Review completed` vs `Review rate limited`), and a rate-limited round often posts nothing else at all: no review, no threads, no comment. Read it with `gh pr checks --json name,state,bucket,description` (the field is not returned by default, and `gh pr view --json statusCheckRollup` drops it entirely) before reporting a PR as reviewed.

---

## Provider Detection

```bash
git remote get-url origin
```

| Remote URL contains                | Provider | CLI    | PR term |
|------------------------------------|----------|--------|---------|
| `github.com`                       | GitHub   | `gh`   | PR      |
| `gitlab.com` or self-hosted GitLab | GitLab   | `glab` | MR      |

If ambiguous or both present, ask the user.

**The `glab` recipes follow GitLab's documentation and are not exercised against a live instance** -- the GitHub ones are. Treat flags and JSON shapes on the GitLab side as documented-but-unverified: check `glab <command> --help` before relying on one in an unattended flow, and prefer `-F json | jq` over assuming a field exists.

---

## Read-Only vs Write Classification

- **Read-only** (safe to auto-approve): `view`, `list`, `status`, `diff`, `checks`, `search`
- **Write** (require user approval): `create`, `edit`, `merge`, `close`, `reopen`, `comment`, `review`, `approve`

> **Reference**: See `references/allowlist.md` for tiered auto-approval patterns.

---

## PR/MR Summary (Current Branch)

**GitHub:**
```bash
gh pr view --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName
```

**GitLab:**
```bash
glab mr view -F json | jq '{iid:.iid,title:.title,state:.state,draft:.draft,merge_status:.merge_status,target:.target_branch,source:.source_branch}'
```

## PR/MR Summary (By Number)

**GitHub:**
```bash
gh pr view {number} --json number,title,state,isDraft,reviewDecision,mergeable,baseRefName,headRefName
```

**GitLab:**
```bash
glab mr view {iid} -F json | jq '{iid:.iid,title:.title,state:.state,draft:.draft,merge_status:.merge_status,target:.target_branch,source:.source_branch}'
```

## Review State

> **Reference**: See `references/review-queries.md` for advanced review queries: per-reviewer state, approval checks, pending reviewers.

**GitHub:**
```bash
gh pr view --json reviews,reviewRequests,latestReviews
```

**GitLab:**
```bash
glab mr view -F json | jq '{upvotes:.upvotes,reviewers:[.reviewers[]?.username]}'
```

For detailed approval info (GitLab):
```bash
glab api projects/{project_id}/merge_requests/{iid}/approvals | jq '{approved:.approved,approvers:[.approved_by[]?.user.username]}'
```

## Files Changed

**GitHub:**
```bash
gh pr diff --name-only
```

**GitLab:**
```bash
glab mr diff
```

## File Stats

**GitHub:**
```bash
gh pr view --json files
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/changes | jq '[.changes[] | {path:.new_path,added:.diff | split("\n") | map(select(startswith("+"))) | length,removed:.diff | split("\n") | map(select(startswith("-"))) | length}]'
```

## List Open PRs/MRs

**GitHub:**
```bash
gh pr list --json number,title,author,reviewDecision,updatedAt
```

**GitLab:**
```bash
glab mr list -F json | jq '[.[] | {iid:.iid,title:.title,author:.author.username,updated:.updated_at}]'
```

## PR/MR Commits

**GitHub:**
```bash
gh pr view --json commits
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/commits | jq '[.[] | {sha:.short_id,title:.title}]'
```

## Search PRs/MRs

**GitHub:**
```bash
gh pr list --search "review-requested:@me" --json number,title,url
```

**GitLab:**
```bash
glab mr list --reviewer=@me -F json | jq '[.[] | {iid:.iid,title:.title,url:.web_url}]'
```

---

## Create PR/MR (Write -- Manual Approval)

| Action            | GitHub                                                 | GitLab                                                    |
|-------------------|--------------------------------------------------------|-----------------------------------------------------------|
| Create draft      | `gh pr create --draft --fill`                          | `glab mr create --draft --fill`                           |
| Create with title | `gh pr create --title "feat: ..." --body "..."`        | `glab mr create --title "feat: ..." --description "..."` |
| Create + request Copilot review | `gh pr create --title "..." --body "..." --reviewer @copilot` (gh >= 2.88) | n/a |

**Keep bodies lean.** The title follows Conventional Commits (`git-commit` skill); the body is a few sentences or tight bullets on what changes and why -- no restating the diff, no boilerplate sections beyond what the repo's PR template requires.

After creating a **non-draft** GitHub PR, check `gh api repos/{owner}/{repo}/pulls/{n}/requested_reviewers` before requesting any bot review -- repos with automatic Copilot review already have one in flight (and it will NOT show in `gh pr view --json reviewRequests`). On a repo you know has **no** auto-review, skip the create-then-edit round-trip and request Copilot at creation with `--reviewer @copilot`; when unsure, create normally and let the detection decide.

## Merge (Write -- Manual Approval)

| Action       | GitHub                                | GitLab                                        |
|--------------|---------------------------------------|-----------------------------------------------|
| Squash merge | `gh pr merge --squash --delete-branch` | `glab mr merge --squash --remove-source-branch` |
| Rebase merge | `gh pr merge --rebase --delete-branch` | `glab mr merge --rebase --remove-source-branch` |

---

## Review Comment Handling

> **Reference**: See `references/pr-comment-workflow.md` for the full opinionated workflow with all command patterns and examples.

The workflow has two distinct phases -- never mix them:

**Phase 1: Analyze and Fix (local work, no GitHub API writes)** -- the fetches are zero-approval; `git commit`/`git push` are writes and stay a human checkpoint
1. Fetch all unresolved review threads in a single GraphQL query with inline `--jq` filter, then the bot's review **bodies** -- threads are not the whole review (see below)
2. For each thread: read the file at the referenced path+line, check if the comment is valid by researching the codebase (patterns, conventions, CLAUDE.md, git log)
3. Be critical -- validate each comment against actual code before accepting. Reviewers can be wrong.
4. Make all necessary code fixes -- without adding code comments that narrate the fix or restate what the code already reads
5. Commit the fixes (as many commits as the change needs) and push **once** for the whole round -- the message describes the change itself, never the review process (no "address review feedback", bot names, or round numbers; see the `git-commit` skill)

**Not every finding is a thread (CodeRabbit).** Nitpicks (under `profile: chill`), outside-diff-range findings, duplicates and failed-to-post comments exist only in the review *body*; `reviewThreads` never returns them, which is the usual reason a review looks handled but isn't. Fetch the bot's review bodies, triage those items too, and reconcile the top-level bot threads against the `Actionable comments posted: N` the reviews claim -- commands in `references/bot-review-loop.md`.

**Phase 2: Reply and Resolve (one batched command, one approval)**
6. Combine all replies and all resolves into a single `&&`-chained command
7. REST replies first, then a single GraphQL mutation with aliases to batch-resolve all handled threads
8. Leave "Needs discussion" threads unresolved

This ordering matters: pushing fixes first ensures reviewers see the changes when they read replies. Never reply to a comment claiming "Fixed" before the fix is actually pushed.

### Fetch Unresolved Threads (Zero Approvals)

**GitHub** -- one command with `$(...)` substitution. **Generate as a single line** and do NOT prepend variable assignments (`OWNER=...`, `REPO=...`) -- both break allowlist matching:

```bash
gh api graphql -f query="{ repository(owner: \"$(gh repo view --json owner --jq '.owner.login')\", name: \"$(gh repo view --json name --jq '.name')\") { pullRequest(number: $(gh pr view --json number --jq '.number')) { reviewThreads(first: 100) { totalCount pageInfo { hasNextPage endCursor } nodes { id isResolved isOutdated path line startLine comments(first: 20) { nodes { id fullDatabaseId body author { login } } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads | {total: .totalCount, nextCursor: (if .pageInfo.hasNextPage then .pageInfo.endCursor else null end), unresolved: [.nodes[] | select(.isResolved==false)]}'
```

Returns the unresolved threads plus the two numbers that prove the fetch was complete. **`reviewThreads` does not paginate on its own** -- a non-null `nextCursor` means threads 101+ exist and may hold unresolved comments; re-run with `reviewThreads(first: 100, after: \"{nextCursor}\")` and merge before deciding anything is handled.

**Response field mapping (critical -- using wrong ID causes silent failures):**

| Field              | Format                  | Use for                      |
|--------------------|-------------------------|------------------------------|
| thread `.id`       | `PRRT_...` (node ID)    | `resolveReviewThread` mutation |
| comment `.fullDatabaseId` | `"2949637341"` (string) | REST reply endpoint          |
| comment `.id`      | `PRRC_...` (node ID)    | Not typically needed          |

**GitLab (REST):**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --paginate | jq '[.[] | select(.notes[0].resolvable==true and .notes[0].resolved==false) | {id:.id,path:.notes[0].position.new_path,line:.notes[0].position.new_line,body:.notes[0].body,author:.notes[0].author.username}]'
```

### Reply and Resolve (One Batched Command)

**GitHub** -- combine all REST replies and a batch GraphQL resolve mutation into one `&&`-chained command. Reply uses `fullDatabaseId` (a string; `databaseId` is deprecated), resolve uses thread `id` (PRRT_ node ID):

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_1}/replies -f body="Fixed in {sha} -- {explanation}" && \
gh api repos/{owner}/{repo}/pulls/{pr}/comments/{fullDatabaseId_2}/replies -f body="Addressed in {sha}" && \
gh api graphql -f query="
mutation {
  t1: resolveReviewThread(input: {threadId: \"PRRT_thread1\"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: \"PRRT_thread2\"}) { thread { isResolved } }
}"
```

**GitLab:**
```bash
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1}/notes --method POST --field "body=Fixed in {sha}" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_1} --method PUT --field "resolved=true" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2}/notes --method POST --field "body=Addressed" && \
glab api projects/{id}/merge_requests/{iid}/discussions/{disc_2} --method PUT --field "resolved=true"
```

---

## Code Review Policy (Convention)

Review preferences are declared in a `Code Review Policy` section of an agent instructions file -- check for one before requesting any review. Two scopes, most specific wins:

1. **Repo policy** -- the project's AGENTS.md or CLAUDE.md. AGENTS.md is the preferred home: Copilot code review itself reads the root AGENTS.md, so one file steers both the agent and the reviewer.
2. **User-global policy** -- the user's global agent instructions (Claude Code: `~/.claude/CLAUDE.md`) as the default flow across repos.

**Both reviewers are optional.** Detect what is actually available (CodeRabbit app installed / CLI authenticated, Copilot ruleset or observed auto-review) and never assume a reviewer exists, is paid for, or should be added. `Reviewer: none` is a valid policy -- human review only. Example:

```markdown
## Code Review Policy
- Reviewer: coderabbit            # coderabbit | copilot | both | none
- CodeRabbit plan: pro+ until 2026-08-20, then pro   # free | pro | pro+ | enterprise
- Copilot billing: legacy         # legacy (premium requests) | credits
- Local review: coderabbit review --committed   # run before every push
- PR review rounds: ask           # ask | loop <= N
```

**Declare the CodeRabbit plan -- nothing reports it.** `coderabbit usage` gives the billing period, not the tier, and the API never mentions it, so an agent that isn't told will either over-schedule PRs into a bucket that cannot serve them or idle a bucket that could. The plan sets the hourly allowance (Free 1 / Pro 5 / Pro+ 10 PR reviews per developer -- table in the `coderabbit` skill), which in turn sets how many PRs can be non-draft at once and how much iteration belongs in the local CLI lane. Note a trial's end date the same way: the queue that works on Pro+ stalls on Pro the morning the trial lapses. **Undeclared**, infer a floor instead of guessing -- `bot_bucket` (`references/bot-review-loop.md`) lists recent rounds across every open PR, and the completions in the trailing hour are a lower bound on the real allowance.

**When no policy exists (either scope), default conservative:** if an auto-review fired, process that round; then ask before any billable re-request -- and inform the recommendation with the quality of the round's findings (mostly valid substantive issues -> another round likely pays off; mostly noise -> stop). Never initiate billable reviews unprompted on repos without auto-review; loop only when explicitly asked.

## Bot Review Loop (GitHub)

> **Reference**: See `references/bot-review-loop.md` for the full loop: per-bot config blocks (Copilot, CodeRabbit), the `bot_status`/`bot_tick` driver, polling, and termination logic.
> **Reference**: See `references/copilot-review-config.md` for Copilot review configuration: billing models and quota checks, auto-review ruleset detection/management, custom instructions, and the Copilot CLI local review option.

GitHub review bots (Copilot, CodeRabbit) are **GitHub-only**, **asynchronous** (~minutes per review), and never block: their review `state` is always `COMMENTED`. The loop is identical per bot; only the **identity** (which login to filter), the **re-request trigger**, and the detectable **failure/rate-limit/clean-review notices** differ -- a per-bot config block sets them. Enablement detection is partial (the Copilot auto-review *ruleset* is readable, but personal-setting auto-review and overall bot availability are not -- see Context Check above), so `bot_tick` exits `5` when the remote isn't GitHub or the re-request errors; an accepted-but-unanswered request exits `3` (slow *or* silently unavailable).

Iterate until no **valid** comments remain. Source a bot's config + the `bot_status`/`bot_tick` driver -- one round is:

1. **Re-request only when needed + wait** -- `bot_tick {N}` first checks for unresolved bot threads (handle those, never re-request over them), then a review at HEAD (a clean CodeRabbit review posts **no review object** -- its "no actionable comments" walkthrough text or a "Review finished." ack is the clean signal, never a failure or rate limit), then HEAD's `CodeRabbit` commit status, which is `success` even when the round was refused (`Review rate limited` = HEAD was never reviewed), then a pending request in REST `requested_reviewers` (auto-review on non-draft PR creation usually means round 1 needs no re-request at all). Only if none of those apply does it re-request (Copilot: `gh pr edit {N} --add-reviewer "@copilot"`, gh >= 2.88, no auto re-review on push; CodeRabbit: a `@coderabbitai review` comment), then polls for the async review. Returns `0` clean / `2` not clean / `3` retry / `4` failed (already cooled down ~5 min + re-requested -- re-run to poll) / `5` not applicable / `6` rate-limited.
2. **Validate, don't blind-fix** -- evaluate each unresolved comment *and* every body-only bucket of the review (nitpick, outside-diff-range, duplicate, failed-to-post; Research Checklist); bots can be out of context or outdated. Fix valid ones (commit each, then one push per round; the message names the change, never the bot or round), reply with a rationale + resolve invalid ones. To clear every reviewer, run the loop once per active bot and handle human threads via the comment workflow above.
3. **Terminate** -- stop when the bot has no comments; on **zero valid comments** (re-requesting would only resurface them); after ~3 consecutive failed reviews (`exit 4` is transient -- e.g. "Copilot encountered an error" -- and `bot_tick` retries it with a ~5-min cooldown + re-request; only *repeated* failure is structural: oversized PR, binary files, quota; escalate); on a rate limit (`exit 6`: CodeRabbit -- schedule from the `BOT_WAIT_UNTIL` the tick exported, since the notice it came from can be edited away; that is the printed window when one was readable, else ~10 min from the bounce, doubling only after a retry bounces again. Check `bot_bucket` first -- a sibling PR reviewed after your bounce means the shared bucket is open now -- then post a **fresh** trigger, since nothing is queued; or skip the bot when Copilot also covers the repo; Copilot -- a hard weekly limit diagnosed from the review run's CI log behind the generic "encountered an error" comment: never re-request before the logged reset date, report cause + date to the user); on unavailability (`exit 5`); after the round cap (default 5, or "loop 3"); or if HEAD is unchanged since the last round.

**Rounds are billable, and on auto-review repos a push is a round** -- each Copilot round bills a full review; each CodeRabbit round spends hourly quota that is **per developer, not per PR**: all your open PRs contend for the same review windows, so with several PRs in flight keep the waiting ones as drafts (excluded from auto-review; marking ready is the request) and promote one at a time -- see the reference's Scheduling Several PRs Through One Bucket and One Push Per Round. Without an explicit loop instruction or a permissive Code Review Policy: process the auto-review round if one fired, then **ask before re-requesting** (recommend based on finding quality). Autonomous looping is for when the user asked for it.

**Identity gotcha:** each bot has a `[bot]`-suffixed login on REST and an unsuffixed one on GraphQL threads (Copilot: `copilot-pull-request-reviewer[bot]` / `copilot-pull-request-reviewer`, plus `Copilot` on REST inline comments; CodeRabbit: `coderabbitai[bot]` / `coderabbitai`). Filter the right one per surface.

---

## Line-Specific Comments (Write)

> **Reference**: See `references/line-comments.md` for full patterns: single-line, multi-line range, replies, edit/delete, batch reviews.

**GitHub:**
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="{comment}" -f path="{file}" -F line={line} -f side=RIGHT \
  -f commit_id="$(gh pr view {pr} --json headRefOid --jq .headRefOid)"
```

**GitLab:**
```bash
glab api projects/{project_id}/merge_requests/{iid}/discussions --method POST \
  --field "body={comment}" \
  --field "position[base_sha]={base_sha}" \
  --field "position[head_sha]={head_sha}" \
  --field "position[start_sha]={base_sha}" \
  --field "position[position_type]=text" \
  --field "position[new_path]={file}" \
  --field "position[new_line]={line}"
```

---

## Read-Only API Endpoints

> **Reference**: See `references/api-readonly.md` for the full list of read-only REST and GraphQL endpoints (PR data, issue timelines, review threads) for both providers.

---

## Key `glab` vs `gh` Differences

| Aspect                | `gh`                                    | `glab`                                      |
|-----------------------|-----------------------------------------|---------------------------------------------|
| MR description flag   | `--body`                                | `--description`                             |
| Delete source branch  | `--delete-branch`                       | `--remove-source-branch`                    |
| JSON output           | `--json field1,field2` (filtered)       | `-F json` (full resource)                   |
| jq filtering          | `--jq '.expr'` (native)                | pipe to `\| jq '.expr'`                    |
| Squash on create      | `--squash`                              | `--squash-before-merge`                     |
| GraphQL               | `gh api graphql -f query='...'`         | Not supported (REST only)                   |
| Thread resolution     | GraphQL mutation                        | `PUT /discussions/:id` with `resolved=true` |
| Approval model        | Review states (APPROVED, CHANGES_REQUESTED) | Approval rules + approve/revoke        |
| Pagination            | `--paginate` (subcommands + api)        | `--paginate` (api only), `-P` (subcommands) |

---

## Allowlist

> **Reference**: See `references/allowlist.md` for tiered `Bash(command:*)` patterns covering all read-only operations -- safe to auto-approve in Claude Code `settings.json` or OpenCode config.
