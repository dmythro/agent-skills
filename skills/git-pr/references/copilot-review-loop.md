# Copilot Review Loop

**Iterate GitHub Copilot code-review rounds on a PR until Copilot has no unresolved comments left on the current HEAD.** This builds on `pr-comment-workflow.md` for the per-round evaluate/fix/reply/resolve work -- this file owns only the Copilot-specific parts: re-requesting, waiting for the async review, detecting failed reviews, and deciding when to stop. Copilot code review is **GitHub-only**, **asynchronous**, and **never blocks** (its review state is always `COMMENTED`), so treat its output as advisory, not a merge gate.

## How Copilot Reviews Appear

Copilot exposes **three different author logins for the same bot**, depending on which surface you query. Filtering the wrong one returns nothing and silently breaks detection:

| Surface             | Endpoint                                         | `login`                              | `type` |
|---------------------|--------------------------------------------------|--------------------------------------|--------|
| Review submission   | `gh api repos/{owner}/{repo}/pulls/{n}/reviews`  | `copilot-pull-request-reviewer[bot]` | `Bot`  |
| Inline comment      | `gh api repos/{owner}/{repo}/pulls/{n}/comments` | `Copilot`                            | `Bot`  |
| Thread (GraphQL)    | `reviewThreads` first comment `author.login`     | `copilot-pull-request-reviewer`      | `Bot`  |

Other facts that shape the loop:

- **State is always `COMMENTED`** -- never `APPROVED` or `CHANGES_REQUESTED`. Do **not** use `reviewDecision` or approval checks to detect Copilot; they never reflect it.
- Each review carries `commit_id` and `submitted_at`. Match `commit_id` against the current HEAD to know Copilot reviewed the latest code.
- A successful review body contains: `Copilot reviewed N out of M changed files in this pull request and generated K comments.` (singular `comment` when `K==1`). Parse the digits, not the word.
- **Reviews can fail.** Instead of a normal review, Copilot may post `Copilot encountered an error and was unable to review this pull request. You can try again by re-requesting a review.` A failed review is **not** "no comments" -- it must be re-requested, not treated as clean. Common causes (PR too large, binary/minified files in the diff, premium-request quota exhausted) often do **not** resolve on retry, so cap the retries.
- **Repo auto-review (if enabled) covers only the first round.** Copilot does **not** re-review automatically when you push -- every later round needs an explicit re-request.

## Re-Request a Review (Write)

```bash
gh pr edit {N} --add-reviewer "@copilot"
```

- Requires `gh >= 2.88`. Write op; see `allowlist.md` "Copilot Re-Request (Opt-In Write)" to auto-approve it, or approve when prompted.
- Issue it in **exactly this canonical form** (quoted `"@copilot"`, flag last) so the allowlist pattern matches.
- Re-requesting while a request is already pending is a safe no-op.
- Remove the request with `gh pr edit {N} --remove-reviewer "@copilot"` (not part of the loop).

## Detect Done (Read-Only)

The authoritative "are we done?" signal is **the count of unresolved review threads opened by Copilot on the current HEAD** -- not the per-review `K` count (which resets every round and ignores resolution state). `copilot_status` reports it, and distinguishes a **failed** review from a clean one. Every command in it is read-only and allowlistable.

```bash
# Usage: copilot_status <PR_NUMBER>
# Read-only. Exit: 0 = clean (reviewed OK, no unresolved Copilot threads)
#                  2 = unresolved Copilot comments remain
#                  3 = no Copilot review for current HEAD yet (pending / not requested)
#                  4 = Copilot review FAILED ("unable to review") -- re-request needed
copilot_status() {
  pr="$1"
  owner="$(gh repo view --json owner --jq '.owner.login')"
  repo="$(gh repo view --json name --jq '.name')"
  head="$(gh pr view "$pr" --json headRefOid --jq '.headRefOid')"
  fail_re='unable to review this pull request|encountered an error'

  # Latest Copilot review whose commit_id == current HEAD. REST /reviews login = ...[bot].
  review="$(gh api "repos/$owner/$repo/pulls/$pr/reviews" --paginate \
    --jq "[.[] | select(.user.login==\"copilot-pull-request-reviewer[bot]\" and .commit_id==\"$head\")] | last")"

  if [ -n "$review" ] && [ "$review" != "null" ]; then
    # Failed review = error notice in the body, NOT a clean result.
    if printf '%s' "$review" | jq -r '.body' | grep -iqE "$fail_re"; then
      echo "Copilot review FAILED on HEAD ${head:0:8} (encountered an error) -- re-request needed"; return 4
    fi
    k="$(printf '%s' "$review" | jq -r '.body' | grep -oE 'generated [0-9]+ comment' | grep -oE '[0-9]+' | head -1)"
    echo "Copilot reviewed HEAD ${head:0:8}; summary says ${k:-0} comment(s)"

    # Unresolved Copilot threads. GraphQL author.login = copilot-pull-request-reviewer
    # (NOT "Copilot" or "...[bot]"). Match case-insensitively on the "copilot" prefix to be safe.
    unresolved="$(gh api graphql -f query="{ repository(owner: \"$owner\", name: \"$repo\") { pullRequest(number: $pr) { reviewThreads(first: 100) { nodes { isResolved path line comments(first: 1) { nodes { author { login } body } } } } } } }" \
      --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false and (.comments.nodes[0].author.login | ascii_downcase | startswith("copilot")))]')"
    n="$(printf '%s' "$unresolved" | jq 'length')"
    if [ "${n:-0}" -eq 0 ]; then echo "No unresolved Copilot threads -- clean."; return 0; fi
    echo "Unresolved Copilot threads ($n):"
    printf '%s' "$unresolved" | jq -r '.[] | "  \(.path):\(.line)  \(.comments.nodes[0].body | gsub("\n";" ") | .[0:90])"'
    return 2
  fi

  # No review for HEAD. The failure can also arrive as a PR (issue) comment -- distinguish
  # "failed" from "still pending" by scanning Copilot comments posted after the HEAD commit.
  since="$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')"
  failed="$(gh api "repos/$owner/$repo/issues/$pr/comments" --paginate \
    --jq "[.[] | select((.user.login|test(\"copilot\";\"i\")) and (.created_at > \"$since\") and (.body|test(\"$fail_re\";\"i\")))] | length")"
  if [ "${failed:-0}" -gt 0 ]; then
    echo "Copilot review FAILED on HEAD ${head:0:8} (error notice in PR comments) -- re-request needed"; return 4
  fi
  echo "No Copilot review yet for HEAD ${head:0:8} (pending or not requested)"; return 3
}
```

## Wait for the Review (Polling)

Reviews are **asynchronous** (typically ~3-11 min after a request) and there is **no `--watch`** for reviews (unlike `gh pr checks --watch`). `copilot_tick` re-requests when needed, polls until an outcome lands, and on a failed review waits ~90s before re-requesting (capped, since structural failures will not self-heal):

```bash
# Usage: copilot_tick <PR_NUMBER>  -- re-request if needed (incl. after a failure), poll for the outcome.
# Wraps copilot_status. Exit: 0 clean / 2 comments / 3 timed out / 4 failed repeatedly (escalate).
copilot_tick() {
  pr="$1"; fails=0
  copilot_status "$pr"; rc=$?
  while [ "$rc" -eq 3 ] || [ "$rc" -eq 4 ]; do
    if [ "$rc" -eq 4 ]; then
      fails=$(( fails + 1 ))
      if [ "$fails" -ge 3 ]; then
        echo "Copilot review failed ${fails}x -- escalate (PR too large? binary files? Copilot quota?)"; return 4
      fi
      echo "Failed review; waiting ~90s before re-requesting (attempt $fails)"; sleep 90
    fi
    gh pr edit "$pr" --add-reviewer "@copilot" >/dev/null     # WRITE; no-op if pending
    deadline=$(( $(date +%s) + 480 ))                         # ~8 min; keep under the 10-min Bash cap
    rc=3
    while [ "$(date +%s)" -lt "$deadline" ]; do
      sleep 30
      copilot_status "$pr"; rc=$?
      [ "$rc" -ne 3 ] && break                                # got 0, 2, or 4
    done
    [ "$rc" -eq 3 ] && { echo "Review did not arrive within timeout -- re-run copilot_tick to keep waiting"; return 3; }
  done
  return "$rc"
}
```

If `copilot_tick` returns `3` (timeout), just run it again -- it re-checks for an existing review before re-requesting, and re-requesting while pending is a no-op, so re-running is safe and idempotent. For a single invocation, set the Bash tool timeout to `>= 540000` ms.

## The Loop

The agent orchestrates the outer loop; `copilot_tick` handles the mechanical round. Termination has several independent triggers so a noisy or failing bot can never trap the agent.

```
INPUT: PR number N; optional MAX_ROUNDS (from "loop 3" -> 3; default: unlimited)
prev_head = ""; round = 0
repeat:
  round += 1
  head = gh pr view N --json headRefOid --jq .headRefOid
  if MAX_ROUNDS set and round > MAX_ROUNDS:
      STOP "ran the requested N round(s); unresolved Copilot comments remain"
  if round > 1 and head == prev_head:
      STOP "no code change since last round -- re-reviewing identical code
            would resurface the same comments"
  prev_head = head
  copilot_tick N:
    exit 0 -> STOP "clean: Copilot has no unresolved comments"
    exit 4 -> STOP "Copilot review keeps failing -- escalate (PR size / binary files / quota)"
    exit 3 -> STOP "review timed out -- retry later"
    exit 2 -> address this round (pr-comment-workflow.md Phase 1/2):
                evaluate each unresolved Copilot thread with the Research
                  Checklist -- bots false-positive often, so be critical
                fix the valid ones -> commit -> push   (advances HEAD)
                reply + resolve every handled thread (one batched command)
                continue
```

**Why it always terminates.** A round only loops again when the agent pushed a fix, which advances HEAD. A round that changes no code (e.g. every comment was a false positive the agent replied to and resolved) leaves HEAD unchanged: the next iteration's `head == prev_head` guard stops it, and even without that guard `copilot_tick` would not re-request (a review for that HEAD already exists) and would return `0` once the handled threads are resolved. Failed reviews are retried at most a few times inside `copilot_tick`, then surface as `exit 4`. The infinite "new thread for the same nit on the same code" loop cannot happen.

**Round count as the "parameter".** Invocations like "loop the Copilot review 3 rounds" set `MAX_ROUNDS=3`; with no number, loop until clean (the HEAD-unchanged guard still prevents runaway).

## Key Gotchas

1. **Three logins for one bot** -- `copilot-pull-request-reviewer[bot]` (REST reviews), `Copilot` (REST comments), `copilot-pull-request-reviewer` (GraphQL threads). Use the right string per endpoint; the `copilot_status` function above does.
2. **State is always `COMMENTED`** -- never wait on `reviewDecision` for Copilot, and Copilot can never block a merge or satisfy a required-approval rule.
3. **A failed review looks like silence** -- "Copilot encountered an error and was unable to review this pull request" is **not** "no comments". Detect the error phrase (in the review body or a PR comment) and re-request; do not treat it as clean. Retrying will not fix a too-large PR, binary files in the diff, or an exhausted quota -- cap retries and escalate.
4. **No auto re-review on push** -- re-request every round after the first.
5. **Async, no `--watch`** -- always poll with a bounded timeout; re-run on timeout.
6. **Resolving without a code change + re-requesting can resurface the same comment** -- gate every new round on HEAD actually changing.
7. **`K==1` uses singular "comment"** -- parse digits (`generated [0-9]+ comment`), not the word.
8. **Match the canonical re-request form** -- `gh pr edit {N} --add-reviewer "@copilot"`, quoted, flag last, or the allowlist entry will not match and it will prompt.
