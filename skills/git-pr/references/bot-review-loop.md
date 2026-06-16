# Bot Review Loop

**Iterate a code-review bot's rounds on a PR until no *valid* comments remain.** The loop is identical for any GitHub review bot (Copilot, CodeRabbit, ...); only a few things differ per bot, all set in a config block -- the **identity** (which login to filter), the **re-request trigger**, and whether the bot posts a detectable **failure** review (`BOT_FAIL_RE`; Copilot does, CodeRabbit doesn't). These bots are GitHub-only, asynchronous (~minutes per review), and non-blocking (reviews are `COMMENTED`), so treat their output as advisory. Per-comment evaluate/fix/reply/resolve mechanics live in `pr-comment-workflow.md`.

## Run It

Source ONE bot's config block (below) together with the two functions, then drive rounds:

```bash
bot_tick <PR>    # re-request if needed + poll once
```

Branch on the exit code per **The Loop**: `0` done, `2` not clean, `3` retry, `4` failed (escalate), `5` not applicable. A tick returns within ~5 min -- run it in the background, or with a Bash timeout `>= 420000` ms, and re-run on `3`. The functions are read-only except the single re-request write.

## Per-Bot Setup

Source ONE block before the functions. Each sets the identity and a `bot_rerequest` command.

```bash
# --- Copilot --- (gh >= 2.88; does NOT auto re-review on push, so re-request every round)
BOT_REVIEW_LOGIN='copilot-pull-request-reviewer[bot]'                  # REST /reviews author
BOT_THREAD_PREFIX='copilot'                                           # GraphQL thread login starts with this
BOT_FAIL_RE='unable to review this pull request|encountered an error'
bot_rerequest() { gh pr edit "$1" --add-reviewer "@copilot"; }

# --- CodeRabbit --- (auto-reviews on push; re-request on demand via a PR comment)
BOT_REVIEW_LOGIN='coderabbitai[bot]'
BOT_THREAD_PREFIX='coderabbit'
BOT_FAIL_RE=''                                                        # no distinct failure-review phrase
bot_rerequest() { gh pr comment "$1" --body "@coderabbitai review"; }
```

Each bot uses a `[bot]`-suffixed login on REST and an unsuffixed one on GraphQL threads (Copilot: `copilot-pull-request-reviewer[bot]` / `copilot-pull-request-reviewer`; CodeRabbit: `coderabbitai[bot]` / `coderabbitai`). Copilot's REST inline comments use a third form (`Copilot`), but the loop keys on review submissions and threads, so the two vars above suffice.

## bot_status (read-only detector)

The authoritative "done?" signal is **zero unresolved threads from this bot**, gated on the bot having reviewed the current HEAD. The thread query is PR-wide (not commit-filtered); only the review-presence check is matched to HEAD.

```bash
# Usage: bot_status <PR_NUMBER>   (read-only; a Per-Bot Setup block must be sourced)
# Exit: 0 clean (no unresolved threads from this bot) | 2 not clean | 3 pending | 4 failed
# Uses gh's {owner}/{repo} placeholders + inline $(...) so each command matches the allowlist
# patterns on its own -- no owner=/repo=/head= assignments (a VAR= prefix breaks matching).
bot_status() {
  pr="$1"

  # Latest bot review whose commit_id == current HEAD. --paginate applies -q/--jq PER PAGE, so use
  # --slurp (an array of pages) | jq and flatten with .[][] to pick the overall latest.
  review="$(gh api repos/{owner}/{repo}/pulls/$pr/reviews --paginate --slurp | jq -c --arg login "$BOT_REVIEW_LOGIN" --arg head "$(gh pr view "$pr" --json headRefOid --jq .headRefOid)" '[.[][] | select(.user.login==$login and .commit_id==$head)] | last')"

  if [ -n "$review" ] && [ "$review" != "null" ]; then
    if [ -n "$BOT_FAIL_RE" ] && printf '%s' "$review" | jq -r '.body' | grep -iqE "$BOT_FAIL_RE"; then
      echo "$BOT_THREAD_PREFIX review FAILED on current HEAD -- re-request needed"; return 4
    fi
    threads="$(gh api graphql -f query="{ repository(owner: \"$(gh repo view --json owner --jq .owner.login)\", name: \"$(gh repo view --json name --jq .name)\") { pullRequest(number: $pr) { reviewThreads(first: 100) { totalCount nodes { isResolved path line comments(first: 1) { nodes { author { login } body } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads')"
    unresolved="$(printf '%s' "$threads" | jq --arg p "$BOT_THREAD_PREFIX" '[.nodes[] | select(.isResolved==false and ((.comments.nodes[0].author.login // "") | ascii_downcase | startswith($p)))]')"
    n="$(printf '%s' "$unresolved" | jq 'length')"
    if [ "${n:-0}" -gt 0 ]; then
      echo "Unresolved $BOT_THREAD_PREFIX threads ($n):"
      printf '%s' "$unresolved" | jq -r '.[] | "  \(.path):\(.line)  \(.comments.nodes[0].body | gsub("\n";" ") | .[0:90])"'
      return 2
    fi
    # Trust "clean" only if we saw every thread: reviewThreads(first: 100) does not paginate.
    [ "$(printf '%s' "$threads" | jq '.totalCount')" -gt 100 ] && { echo "Inconclusive: >100 threads exceed the 100 fetched"; return 2; }
    echo "No unresolved $BOT_THREAD_PREFIX threads -- clean."; return 0
  fi

  # No review at HEAD. If the bot signals failures via a PR comment (BOT_FAIL_RE set), tell "failed"
  # apart from "still pending". (A force-push that rewrites committer dates can delay this a tick.)
  if [ -n "$BOT_FAIL_RE" ]; then
    failed="$(gh api repos/{owner}/{repo}/issues/$pr/comments --paginate --slurp | jq --arg since "$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')" --arg p "$BOT_THREAD_PREFIX" --arg fail "$BOT_FAIL_RE" '[.[][] | select((.user.login|ascii_downcase|startswith($p)) and (.created_at > $since) and (.body|test($fail;"i")))] | length')"
    [ "${failed:-0}" -gt 0 ] && { echo "$BOT_THREAD_PREFIX review FAILED (error notice in PR comments) -- re-request needed"; return 4; }
  fi
  echo "No $BOT_THREAD_PREFIX review yet for current HEAD (pending or not requested)"; return 3
}
```

## bot_tick (re-request + one bounded poll)

If there's already an outcome at HEAD, return it; otherwise re-request once and poll until the review lands or a 5-minute deadline. **One re-request, one bounded poll, no internal retry loop** -- so a single invocation stays well under the 10-minute Bash cap; the agent's loop owns retries and escalation.

```bash
# Usage: bot_tick <PR_NUMBER>   (one re-request + one bounded poll; needs the config block + bot_rerequest)
# Exit: 0 clean | 2 not clean | 3 pending/timed out | 4 failed | 5 unavailable (non-GitHub or re-request failed)
bot_tick() {
  pr="$1"
  case "$(git remote get-url origin 2>/dev/null)" in
    *github.com*) : ;;
    *) echo "GitHub-only: origin is not github.com"; return 5 ;;
  esac
  bot_status "$pr"; rc=$?
  [ "$rc" -ne 3 ] && return "$rc"            # already have an outcome (0 / 2 / 4)

  if ! bot_rerequest "$pr" >/dev/null 2>&1; then
    echo "Re-request failed -- bot not enabled here, or wrong command (see Per-Bot Setup)"; return 5
  fi
  deadline=$(( $(date +%s) + 300 ))          # 5 min; one invocation stays well under the 10-min Bash cap
  while [ "$(date +%s)" -lt "$deadline" ]; do
    sleep 30; bot_status "$pr"; rc=$?
    [ "$rc" -ne 3 ] && return "$rc"
  done
  echo "No review within this tick -- re-run bot_tick (safe: it re-checks before re-requesting)"; return 3
}
```

## The Loop

The agent drives rounds; `bot_tick` is one mechanical round. **Validate every comment** -- bots are useful but can be out of context or working from outdated knowledge, so judge each on its merits and never blind-fix.

```text
INPUT: PR N; a Per-Bot Setup block sourced; MAX_ROUNDS = integer from the request ("loop 3" -> 3), else 20
prev_head = ""; round = 0
repeat:
  round += 1;  if round > MAX_ROUNDS: STOP "hit round cap -- escalate"
  head = gh pr view N --json headRefOid --jq .headRefOid
  if round > 1 and head == prev_head: STOP "no code change -- re-review would resurface the same points"
  prev_head = head
  bot_tick N:
    0 -> STOP "clean -- this bot has no unresolved comments"
    5 -> STOP "not applicable -- non-GitHub remote, or the bot isn't enabled here"
    3 -> re-run bot_tick (still pending); after a couple of timeouts STOP "timed out / maybe unavailable"
    4 -> STOP "review failed -- likely an oversized PR, binary/minified files, or quota; fix the cause, then re-run"
    2 -> not clean. If bot_status flagged the rare >100-thread inconclusive case, paginate/resolve to
         confirm before trusting clean. Otherwise validate each unresolved comment (pr-comment-workflow.md):
           valid   -> fix
           invalid -> reply with the rationale + resolve (no code change)
         if NONE were valid (nothing to fix): STOP "zero valid comments -- re-requesting would only resurface them"
         else: commit + push (advances HEAD; auto-review bots re-review on the push); re-request this bot; continue
```

**It always terminates.** A round continues only when a *valid* comment was fixed (advancing HEAD); an all-rejected round stops at the zero-valid exit (the `head == prev_head` guard is a backstop). Failure and unavailability stop immediately, and `MAX_ROUNDS` caps the whole thing -- so it converges on substance.

To clear *every* reviewer in one pass, run the loop once per active bot (and address human threads via `pr-comment-workflow.md`); each bot's `bot_status` only counts its own threads.

## How Bot Reviews Appear

- **State is always `COMMENTED`** -- these bots never approve, request changes, block a merge, or satisfy a required-approval rule. Don't use `reviewDecision` to detect them.
- **Identity differs by surface and by bot** -- a `[bot]`-suffixed login on REST, unsuffixed on GraphQL threads (see Per-Bot Setup). Filter the wrong one and detection silently returns nothing.
- **Copilot specifics** -- requires gh >= 2.88; does **not** auto re-review on push (re-request every round; repo auto-review covers only round 1); a failed review reads `Copilot encountered an error and was unable to review this pull request` (detect via `BOT_FAIL_RE`, never treat as clean); the summary says `... generated K comments` (singular at `K==1`).
- **CodeRabbit specifics** -- auto-reviews on each push, so a push usually re-triggers it; re-request on demand with a `@coderabbitai review` PR comment. Resolve its threads like any other (or use its `@coderabbitai resolve` command).

## Preconditions

- **GitHub only** -- these are GitHub review bots; `bot_tick` checks the remote and exits `5`.
- **The bot must be enabled** -- Copilot code review needs a plan that includes it plus org/enterprise + repo settings (Settings > Copilot > Code review); CodeRabbit needs its GitHub App installed on the repo. No read-only "is it enabled" endpoint: a re-request that errors (exit `5`) is the signal; if requests are accepted but no review arrives, that's exit `3` (slow *or* silently unavailable).
- **Allowlist** -- the read-only polls and the opt-in re-request have patterns in `allowlist.md`. Keep REST paths unquoted (`gh api repos/...`) and issue the re-request in the canonical form, or matching fails and it prompts.

## Autonomy & allowlisting

For unattended loops the commands must match the patterns in `allowlist.md`:

- **Read-only detection + the opt-in re-request are allowlist-matchable.** The driver uses `gh api repos/{owner}/{repo}/...` placeholders and inline `$(...)` -- never `owner=`/`repo=`/`head=` assignments (a `VAR=` prefix stops matching) -- and keeps REST paths unquoted, so each command auto-approves under the documented patterns.
- **The helper functions are one shell invocation.** A permission check applies to the whole `bot_status`/`bot_tick` call, not each inner command, so in prompt-mode you approve once per call. For a fully hands-off loop, run in an auto-approve permission mode (or invoke the individual commands, which match the patterns).
- **The per-round writes are the deliberate gate.** Addressing comments -- reply, resolve thread, `git commit`/`push` -- is not auto-approved by default; that's the human checkpoint. Allow those too (or use an auto-approve mode) for end-to-end autonomy.

## Key Gotchas

1. **Identity differs by surface and bot** (Per-Bot Setup) -- the most common detection bug; filter the right login.
2. **A failed review looks like "no comments"** (Copilot) -- detect the error phrase and re-request; never treat it as clean.
3. **Stop on zero *valid* comments, not zero comments** -- reject out-of-context/outdated points with a rationale + resolve; don't blind-fix to silence a bot, and don't chase one that resurfaces rejected points.
