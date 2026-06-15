# Copilot Review Loop

**Iterate GitHub Copilot code-review rounds on a PR until no *valid* comments remain.** Copilot code review is GitHub-only, asynchronous (~3-11 min per request), and never blocking -- its review state is always `COMMENTED`, so treat it as advisory. This file owns the Copilot-specific mechanics (re-request, wait, detect, decide when to stop); it delegates the per-round evaluate/fix/reply/resolve work to `pr-comment-workflow.md`.

## Run It

Source the two functions below, then drive rounds. One mechanical round:

```bash
copilot_tick <PR>    # re-request if needed + poll once
```

Branch on its exit code per **The Loop**: `0` done, `2` validate + fix, `3` retry, `4` failed (escalate), `5` not applicable. A tick returns within ~5 min -- run it in the background, or with a Bash timeout `>= 330000` ms, and just re-run it on `3`. The agent owns the round loop and the validate/fix decision; both functions are read-only except the single re-request write.

## copilot_status (read-only detector)

The authoritative "done?" signal is the count of **unresolved Copilot review threads** (PR-wide -- the thread query is not commit-filtered; only the review-presence check is matched to HEAD), not the per-review `K` count.

```bash
# Usage: copilot_status <PR_NUMBER>   (read-only)
# Exit: 0 clean | 2 not clean (comments, or >100-thread inconclusive) | 3 pending | 4 failed
copilot_status() {
  pr="$1"
  owner="$(gh repo view --json owner --jq '.owner.login')"
  repo="$(gh repo view --json name --jq '.name')"
  head="$(gh pr view "$pr" --json headRefOid --jq '.headRefOid')"
  fail_re='unable to review this pull request|encountered an error'

  # Latest Copilot review at current HEAD. --paginate applies -q/--jq PER PAGE, so use --slurp
  # (an array of pages) | jq and flatten with .[][]. Login on REST /reviews is ...[bot].
  review="$(gh api repos/$owner/$repo/pulls/$pr/reviews --paginate --slurp | jq -c --arg head "$head" '[.[][] | select(.user.login=="copilot-pull-request-reviewer[bot]" and .commit_id==$head)] | last')"

  if [ -n "$review" ] && [ "$review" != "null" ]; then
    if printf '%s' "$review" | jq -r '.body' | grep -iqE "$fail_re"; then
      echo "Copilot review FAILED on HEAD ${head:0:8} -- re-request needed"; return 4
    fi
    k="$(printf '%s' "$review" | jq -r '.body' | grep -oE 'generated [0-9]+ comment' | grep -oE '[0-9]+' | head -1)"
    echo "Copilot reviewed HEAD ${head:0:8}; summary says ${k:-0} comment(s)"
    # Unresolved Copilot threads. GraphQL author.login is copilot-pull-request-reviewer (NOT the
    # ...[bot] or "Copilot" forms -- see the logins table). Match the "copilot" prefix to be safe.
    threads="$(gh api graphql -f query="{ repository(owner: \"$owner\", name: \"$repo\") { pullRequest(number: $pr) { reviewThreads(first: 100) { totalCount nodes { isResolved path line comments(first: 1) { nodes { author { login } body } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads')"
    unresolved="$(printf '%s' "$threads" | jq '[.nodes[] | select(.isResolved==false and (.comments.nodes[0].author.login | ascii_downcase | startswith("copilot")))]')"
    n="$(printf '%s' "$unresolved" | jq 'length')"
    if [ "${n:-0}" -gt 0 ]; then
      echo "Unresolved Copilot threads ($n):"
      printf '%s' "$unresolved" | jq -r '.[] | "  \(.path):\(.line)  \(.comments.nodes[0].body | gsub("\n";" ") | .[0:90])"'
      return 2
    fi
    # Trust "clean" only if we saw every thread: reviewThreads(first: 100) does not paginate.
    [ "$(printf '%s' "$threads" | jq '.totalCount')" -gt 100 ] && { echo "Inconclusive: >100 threads exceed the 100 fetched"; return 2; }
    echo "No unresolved Copilot threads -- clean."; return 0
  fi

  # No review at HEAD: a failure can also arrive as a PR comment. Scan Copilot comments since the
  # HEAD commit. (A force-push that rewrites committer dates can delay this detection to the next tick.)
  since="$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')"
  failed="$(gh api repos/$owner/$repo/issues/$pr/comments --paginate --slurp | jq --arg since "$since" --arg fail "$fail_re" '[.[][] | select((.user.login|test("copilot";"i")) and (.created_at > $since) and (.body|test($fail;"i")))] | length')"
  [ "${failed:-0}" -gt 0 ] && { echo "Copilot review FAILED (error notice in PR comments) -- re-request needed"; return 4; }
  echo "No Copilot review yet for HEAD ${head:0:8} (pending or not requested)"; return 3
}
```

## copilot_tick (re-request + one bounded poll)

If there's already an outcome at HEAD, return it; otherwise re-request once and poll until the review lands or a 5-minute deadline. **One re-request, one bounded poll, no internal retry loop** -- so a single invocation stays well under the 10-minute Bash cap, and the agent's loop (below) owns retries and escalation.

```bash
# Usage: copilot_tick <PR_NUMBER>   (one re-request + one bounded poll)
# Exit: 0 clean | 2 not clean | 3 pending/timed out | 4 failed | 5 unavailable (non-GitHub or not enabled)
copilot_tick() {
  pr="$1"
  case "$(git remote get-url origin 2>/dev/null)" in
    *github.com*) : ;;
    *) echo "GitHub-only: origin is not github.com"; return 5 ;;
  esac
  copilot_status "$pr"; rc=$?
  [ "$rc" -ne 3 ] && return "$rc"           # already have an outcome (0 / 2 / 4)

  # Pending: re-request once (WRITE; no-op if already pending). A failure here means Copilot review
  # isn't enabled for this repo/account (or gh < 2.88) -- stop, don't poll for a review that won't come.
  if ! gh pr edit "$pr" --add-reviewer "@copilot" >/dev/null 2>&1; then
    echo "Cannot request Copilot review -- not enabled here, or gh < 2.88 (plan / org / repo Settings > Copilot)"; return 5
  fi
  deadline=$(( $(date +%s) + 300 ))         # 5 min; one invocation stays well under the 10-min Bash cap
  while [ "$(date +%s)" -lt "$deadline" ]; do
    sleep 30; copilot_status "$pr"; rc=$?
    [ "$rc" -ne 3 ] && return "$rc"
  done
  echo "No review within this tick -- re-run copilot_tick (safe: it re-checks before re-requesting)"; return 3
}
```

## The Loop

The agent drives rounds; `copilot_tick` is one mechanical round. **Validate every comment** -- Copilot is useful but can be out of context or working from outdated knowledge, so judge each on its merits and never blind-fix.

```
INPUT: PR N; MAX_ROUNDS = integer parsed from the request ("loop 3" -> 3), else 20
prev_head = ""; round = 0
repeat:
  round += 1;  if round > MAX_ROUNDS: STOP "hit round cap -- escalate"
  head = gh pr view N --json headRefOid --jq .headRefOid
  if round > 1 and head == prev_head: STOP "no code change -- re-review would resurface the same points"
  prev_head = head
  copilot_tick N:
    0 -> STOP "clean -- Copilot has no unresolved comments"
    5 -> STOP "not applicable -- non-GitHub remote, or Copilot review not enabled"
    3 -> re-run copilot_tick (still pending); after a couple of timeouts STOP "timed out / maybe silently unavailable"
    4 -> STOP "review failed -- likely an oversized PR, binary/minified files, or exhausted quota.
              These rarely resolve on retry; fix the cause, then re-run the loop (or re-request manually)."
    2 -> validate each unresolved Copilot comment (pr-comment-workflow.md Research Checklist):
           valid   -> fix
           invalid -> reply with the rationale + resolve (no code change)
         if NONE were valid (nothing to fix): STOP "zero valid comments -- re-requesting would only resurface them"
         else: commit + push the fixes (advances HEAD); continue
```

**It always terminates.** A round continues only when a *valid* comment was fixed (advancing HEAD); an all-rejected round stops at the zero-valid exit (the `head == prev_head` guard is a backstop). Failures and unavailability stop immediately, and `MAX_ROUNDS` caps the whole thing. So it converges on substance -- ending when Copilot is silent, when only rejectable noise remains, on failure/unavailability, or at the cap.

## How Copilot Reviews Appear

The same bot has **three different logins** by surface -- filter the wrong one and detection silently returns nothing:

| Surface           | Endpoint                                  | `login`                              |
|-------------------|-------------------------------------------|--------------------------------------|
| Review submission | REST `pulls/{n}/reviews`                  | `copilot-pull-request-reviewer[bot]` |
| Inline comment    | REST `pulls/{n}/comments`                 | `Copilot`                            |
| Thread            | GraphQL `reviewThreads` (first comment)   | `copilot-pull-request-reviewer`      |

- **State is always `COMMENTED`** -- Copilot never approves, requests changes, blocks a merge, or satisfies a required-approval rule. Don't use `reviewDecision` to detect it.
- A successful review body says `Copilot reviewed N out of M changed files ... and generated K comments.` (singular at `K==1` -- parse the digits).
- **A failed review is not silence:** `Copilot encountered an error and was unable to review this pull request.` Detect the phrase (review body or PR comment) and re-request; never treat it as clean.
- **No auto re-review on push** -- re-request every round after the first (repo auto-review, if enabled, covers only round 1).

## Preconditions

- **GitHub only** -- no GitLab/`glab` equivalent; `copilot_tick` checks the remote and exits `5`.
- **gh >= 2.88** -- for `--add-reviewer "@copilot"`.
- **Copilot code review enabled** -- needs a plan that includes it, plus enablement at the org/enterprise level and in repo Settings > Copilot > Code review. There is no read-only "is it enabled" endpoint: a re-request that errors (exit `5`) is the signal. If requests are accepted but no review ever arrives, that's exit `3` -- slow *or* silently unavailable, so don't read `3` as merely "slow".
- **Allowlist** -- the read-only polls and the opt-in re-request have patterns in `allowlist.md`. Issue the re-request in the canonical form `gh pr edit {N} --add-reviewer "@copilot"`, and keep REST paths unquoted (`gh api repos/...`), or matching fails and it prompts.

## Key Gotchas

1. **Three logins for one bot** (table above) -- the most common detection bug; filter the right one per surface.
2. **A failed review looks like "no comments"** -- detect the error phrase and re-request; never treat it as clean.
3. **Stop on zero *valid* comments, not zero comments** -- reject out-of-context/outdated points with a rationale + resolve; don't blind-fix to silence Copilot, and don't chase a bot that resurfaces rejected points.
