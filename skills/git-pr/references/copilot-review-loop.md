# Copilot Review Loop

**Iterate GitHub Copilot code-review rounds on a PR until Copilot has no unresolved comments left on the current HEAD.** This builds on `pr-comment-workflow.md` for the per-round evaluate/fix/reply/resolve work -- this file owns only the Copilot-specific parts: re-requesting, waiting for the async review, detecting failed reviews, and deciding when to stop. Copilot code review is **GitHub-only**, **asynchronous**, and **never blocks** (its review state is always `COMMENTED`), so treat its output as advisory, not a merge gate.

## Preconditions

The loop is **GitHub-only** and only works where Copilot code review is enabled:

- **GitHub only** -- Copilot code review is a GitHub feature with no GitLab/`glab` equivalent. Confirm the remote first (`git remote get-url origin` contains `github.com`); for GitLab MRs this loop does not apply. `copilot_tick` checks this and exits `5`.
- **gh >= 2.88** -- earlier versions lack `--add-reviewer "@copilot"`.
- **Copilot code review enabled** -- not available on every account/repo. It needs a plan that includes Copilot code review plus enablement at the org/enterprise level and in repo Settings > Copilot > Code review. There is no read-only "is it enabled" endpoint, so detect it behaviorally: if `gh pr edit {N} --add-reviewer "@copilot"` errors (caught as exit `5`), or no review and no failure notice ever arrive, treat it as unavailable and stop -- do not poll forever.
- **Allowlisting** -- the read-only commands match the patterns in `allowlist.md`; the opt-in re-request needs its own pattern (also in `allowlist.md`). Use unquoted REST paths so they match.

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
#                  2 = not clean -- unresolved Copilot comments, or inconclusive (>100 threads)
#                  3 = no Copilot review for current HEAD yet (pending / not requested)
#                  4 = Copilot review FAILED ("unable to review") -- re-request needed
copilot_status() {
  pr="$1"
  owner="$(gh repo view --json owner --jq '.owner.login')"
  repo="$(gh repo view --json name --jq '.name')"
  head="$(gh pr view "$pr" --json headRefOid --jq '.headRefOid')"
  fail_re='unable to review this pull request|encountered an error'

  # Latest Copilot review whose commit_id == current HEAD. REST /reviews login = ...[bot].
  # NOTE: --paginate with -q/--jq applies the filter PER PAGE (one result per page). Use
  # --paginate --slurp (an array of pages) piped to jq and flatten with .[][], so "last" is
  # the overall latest across all pages -- not a per-page last. (--slurp can't combine with -q.)
  review="$(gh api repos/$owner/$repo/pulls/$pr/reviews --paginate --slurp | jq -c --arg head "$head" '[.[][] | select(.user.login=="copilot-pull-request-reviewer[bot]" and .commit_id==$head)] | last')"

  if [ -n "$review" ] && [ "$review" != "null" ]; then
    # Failed review = error notice in the body, NOT a clean result.
    if printf '%s' "$review" | jq -r '.body' | grep -iqE "$fail_re"; then
      echo "Copilot review FAILED on HEAD ${head:0:8} (encountered an error) -- re-request needed"; return 4
    fi
    k="$(printf '%s' "$review" | jq -r '.body' | grep -oE 'generated [0-9]+ comment' | grep -oE '[0-9]+' | head -1)"
    echo "Copilot reviewed HEAD ${head:0:8}; summary says ${k:-0} comment(s)"

    # Unresolved Copilot threads. GraphQL author.login = copilot-pull-request-reviewer
    # (NOT "Copilot" or "...[bot]"). Match case-insensitively on the "copilot" prefix to be safe.
    threads="$(gh api graphql -f query="{ repository(owner: \"$owner\", name: \"$repo\") { pullRequest(number: $pr) { reviewThreads(first: 100) { totalCount nodes { isResolved path line comments(first: 1) { nodes { author { login } body } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads')"
    unresolved="$(printf '%s' "$threads" | jq '[.nodes[] | select(.isResolved==false and (.comments.nodes[0].author.login | ascii_downcase | startswith("copilot")))]')"
    n="$(printf '%s' "$unresolved" | jq 'length')"
    if [ "${n:-0}" -gt 0 ]; then
      echo "Unresolved Copilot threads ($n):"
      printf '%s' "$unresolved" | jq -r '.[] | "  \(.path):\(.line)  \(.comments.nodes[0].body | gsub("\n";" ") | .[0:90])"'
      return 2
    fi
    # Only trust "clean" if we actually fetched every thread -- the query caps at 100 and does
    # not paginate, so a PR with >100 threads could hide unresolved ones beyond the first page.
    total="$(printf '%s' "$threads" | jq '.totalCount')"
    if [ "${total:-0}" -gt 100 ]; then
      echo "Inconclusive: $total review threads exceed the 100 fetched -- resolve threads or paginate before trusting 'clean'"; return 2
    fi
    echo "No unresolved Copilot threads -- clean."; return 0
  fi

  # No review for HEAD. The failure can also arrive as a PR (issue) comment -- distinguish
  # "failed" from "still pending" by scanning Copilot comments posted after the HEAD commit.
  since="$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')"
  failed="$(gh api repos/$owner/$repo/issues/$pr/comments --paginate --slurp | jq --arg since "$since" --arg fail "$fail_re" '[.[][] | select((.user.login|test("copilot";"i")) and (.created_at > $since) and (.body|test($fail;"i")))] | length')"
  if [ "${failed:-0}" -gt 0 ]; then
    echo "Copilot review FAILED on HEAD ${head:0:8} (error notice in PR comments) -- re-request needed"; return 4
  fi
  echo "No Copilot review yet for HEAD ${head:0:8} (pending or not requested)"; return 3
}
```

## Wait for the Review (Polling)

Reviews are **asynchronous** (typically ~3-11 min after a request) and there is **no `--watch`** for reviews (unlike `gh pr checks --watch`). `copilot_tick` re-requests when needed, polls until an outcome lands, and on a failed review waits ~90s before re-requesting (capped at **3 errored reviews**, since structural failures will not self-heal):

```bash
# Usage: copilot_tick <PR_NUMBER>  -- re-request if needed (incl. after a failure), poll for the outcome.
# Wraps copilot_status. Exit: 0 clean / 2 not clean (comments or inconclusive) / 3 timed out / 4 failed repeatedly / 5 unavailable (non-GitHub, or Copilot review not enabled).
copilot_tick() {
  pr="$1"; fails=0
  # GitHub only -- Copilot code review has no GitLab equivalent.
  case "$(git remote get-url origin 2>/dev/null)" in
    *github.com*) : ;;
    *) echo "Copilot review loop is GitHub-only (origin is not github.com)"; return 5 ;;
  esac
  copilot_status "$pr"; rc=$?
  while [ "$rc" -eq 3 ] || [ "$rc" -eq 4 ]; do
    if [ "$rc" -eq 4 ]; then
      fails=$(( fails + 1 ))
      if [ "$fails" -ge 3 ]; then
        echo "Copilot review failed ${fails}x -- escalate (PR too large? binary files? Copilot quota?)"; return 4
      fi
      echo "Failed review; waiting ~90s before re-requesting (attempt $fails)"; sleep 90
    fi
    # Re-request (WRITE; no-op if pending). A failure usually means Copilot code review is not
    # enabled here (or gh < 2.88) -- stop rather than poll for a review that will never come.
    if ! gh pr edit "$pr" --add-reviewer "@copilot" >/dev/null 2>&1; then
      echo "Cannot request Copilot review -- not enabled here, or gh < 2.88 (check plan / org / repo Settings > Copilot > Code review)"; return 5
    fi
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
INPUT: PR number N; MAX_ROUNDS (from "loop 3" -> 3; default 20)
prev_head = ""; round = 0
repeat:
  round += 1
  head = gh pr view N --json headRefOid --jq .headRefOid
  if round > MAX_ROUNDS:
      STOP "hit the round cap (MAX_ROUNDS, default 20); comments may remain -- escalate"
  if round > 1 and head == prev_head:
      STOP "no code change since last round -- re-reviewing identical code
            would resurface the same comments"
  prev_head = head
  copilot_tick N:
    exit 0 -> STOP "clean: Copilot has no unresolved comments"
    exit 5 -> STOP "not applicable -- non-GitHub remote, or Copilot code review not enabled here"
    exit 4 -> STOP "Copilot review keeps failing -- escalate (PR size / binary files / quota)"
    exit 3 -> STOP "review timed out -- retry later"
    exit 2 -> address this round (pr-comment-workflow.md Phase 1/2):
                evaluate each unresolved Copilot thread with the Research
                  Checklist -- bots false-positive often, so be critical
                fix the valid ones -> commit -> push   (advances HEAD)
                reply + resolve every handled thread (one batched command)
                continue
```

**Why it always terminates.** A round only loops again when the agent pushed a fix, which advances HEAD. A round that changes no code (e.g. every comment was a false positive the agent replied to and resolved) leaves HEAD unchanged: the next iteration's `head == prev_head` guard stops it, and even without that guard `copilot_tick` would not re-request (a review for that HEAD already exists) and would return `0` once the handled threads are resolved. Failed reviews are retried at most **3 times** inside `copilot_tick` before surfacing as `exit 4`, and the whole loop is bounded by `MAX_ROUNDS` (default 20). The infinite "new thread for the same nit on the same code" loop cannot happen.

**Round count as the "parameter".** Invocations like "loop the Copilot review 3 rounds" set `MAX_ROUNDS=3`; with no number it defaults to **20**. The HEAD-unchanged guard and the failed-review cap usually stop the loop well before the cap.

## Key Gotchas

1. **Three logins for one bot** -- `copilot-pull-request-reviewer[bot]` (REST reviews), `Copilot` (REST comments), `copilot-pull-request-reviewer` (GraphQL threads). Use the right string per endpoint; the `copilot_status` function above does.
2. **State is always `COMMENTED`** -- never wait on `reviewDecision` for Copilot, and Copilot can never block a merge or satisfy a required-approval rule.
3. **A failed review looks like silence** -- "Copilot encountered an error and was unable to review this pull request" is **not** "no comments". Detect the error phrase (in the review body or a PR comment) and re-request; do not treat it as clean. Retrying will not fix a too-large PR, binary files in the diff, or an exhausted quota -- cap retries and escalate.
4. **No auto re-review on push** -- re-request every round after the first.
5. **Async, no `--watch`** -- always poll with a bounded timeout; re-run on timeout.
6. **Resolving without a code change + re-requesting can resurface the same comment** -- gate every new round on HEAD actually changing.
7. **`K==1` uses singular "comment"** -- parse digits (`generated [0-9]+ comment`), not the word.
8. **Match the canonical re-request form** -- `gh pr edit {N} --add-reviewer "@copilot"`, quoted, flag last, or the allowlist entry will not match and it will prompt.
9. **The thread query caps at 100** -- `reviewThreads(first: 100)` is not paginated. `copilot_status` compares against `totalCount` and refuses to report "clean" when more threads exist than were fetched, so the termination guarantee holds; paginate (cursor `after:`) only if you routinely exceed 100 threads on one PR.
10. **GitHub-only, and not always available** -- no GitLab equivalent, and Copilot code review needs an enabled plan/org/repo. Gate on the provider and on the re-request succeeding (exit `5`); don't poll for a review that will never come. There is no read-only availability endpoint -- the re-request erroring is the signal.
11. **Use unquoted REST paths** -- `gh api repos/$owner/...` (not `gh api "repos/..."`); the quotes prevent the unquoted allowlist patterns from matching and trigger a prompt.
