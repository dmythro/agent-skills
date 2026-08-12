# Bot Review Loop

**Iterate a code-review bot's rounds on a PR until no *valid* comments remain.** The loop is identical for any GitHub review bot (Copilot, CodeRabbit, ...); only a few things differ per bot, all set in a config block -- the **identity** (which login to filter), the **re-request trigger**, and which non-review outcomes it can post: a **failure** notice (`BOT_FAIL_RE`; Copilot does, CodeRabbit doesn't), a **rate-limit** notice (`BOT_LIMIT_RE`; CodeRabbit does, Copilot doesn't -- Copilot's rate limits hide behind its generic failure notice, and only the review run's CI log tells them apart: `bot_fail_diag` below), or a **clean-review** notice (`BOT_CLEAN_RE`; CodeRabbit does -- a clean CodeRabbit review submits **no review object at all**, only comments; Copilot always submits one). These bots are GitHub-only, asynchronous (~minutes per review), and non-blocking (reviews are `COMMENTED`), so treat their output as advisory. Per-comment evaluate/fix/reply/resolve mechanics live in `pr-comment-workflow.md`.

**Every round costs something -- the loop is a budget, not a free retry.** Each Copilot review (including every re-request) bills fully: 13 premium requests on legacy annual plans (Pro: 300/month = up to ~23 reviews, shared with all other premium features), or token-metered AI credits plus GitHub Actions minutes on current plans (see `copilot-review-config.md` for the models and quota checks). Each CodeRabbit review spends an hourly per-plan bucket. Default posture without an explicit loop instruction or permissive Code Review Policy (SKILL.md): process the auto-review round if one fired, then **ask before any billable re-request**, recommending based on the round's finding quality -- mostly valid substantive findings suggests another round pays off; mostly rejected noise suggests stopping. Cheap iteration belongs *before* the PR: local CodeRabbit CLI reviews (`coderabbit` skill) draw from a separate hourly bucket than PR reviews (though usage-based *overage* credits, where enabled, are a shared pool).

**Auto-review changes round 1 -- where it exists.** Automatic Copilot code review comes from either a repo/org **ruleset** (the `copilot_code_review` branch rule -- readable via `gh api repos/{owner}/{repo}/rules/branches/{branch}`, with `review_on_push` and `review_draft_pull_requests` parameters; recipes in `copilot-review-config.md`) or the PR author's **personal Copilot setting** (no API, invisible) -- so a clean ruleset check still doesn't mean "off"; detect the actual state. Note the ruleset endpoint returns 403 on Free-plan private repos -- treat that as unknown, not disabled. When enabled, Copilot is requested automatically on every **non-draft** PR at creation and when a draft is marked ready (plus on each push only if `review_on_push` is set). CodeRabbit similarly auto-reviews new PRs and pushes once its app is installed. The driver handles both worlds without configuration: `bot_status` reports outstanding comments before anything else, and `bot_tick` never re-requests while the bot already has a pending review request (`bot_requested`, REST `requested_reviewers`) -- so on auto-review repos the first round typically issues **no request at all** (it waits for or finds the automatic one), and on repos without it the same tick falls through to a normal single re-request.

## Run It

Source ONE bot's config block (below) together with the three functions (`bot_status`, `bot_requested`, `bot_tick`), then drive rounds:

```bash
bot_tick <PR>    # re-request if needed + poll once
```

Branch on the exit code per **The Loop**: `0` done, `2` not clean, `3` retry, `4` failed (the tick already cooled down ~5 min and re-requested -- re-run to poll; escalate only when it repeats), `5` not applicable, `6` rate-limited (CodeRabbit: wait out the window or defer to another bot; Copilot hard limit from the CI log: STOP and report the reset date -- never re-request). A tick returns within ~6 min -- run it in the background, or with a Bash timeout `>= 420000` ms, and re-run on `3`. The functions are read-only except the single re-request write.

## Per-Bot Setup

Source ONE block before the functions. Each sets the identity and a `bot_rerequest` command.

```bash
# --- Copilot --- (gh >= 2.88; does NOT auto re-review on push, so re-request every round.
#     Auto-review repos request it on non-draft PR creation / draft->ready: it then sits in
#     REST requested_reviewers as login 'Copilot' until the review lands -- NOT visible in
#     gh pr view --json reviewRequests, which omits bot reviewers. On repos WITHOUT auto-review,
#     round 1 can be seeded at creation: gh pr create --reviewer @copilot.)
BOT_REVIEW_LOGIN='copilot-pull-request-reviewer[bot]'                  # REST /reviews author
BOT_THREAD_PREFIX='copilot'                                           # GraphQL thread login starts with this
BOT_FAIL_RE='unable to review this pull request|encountered an error'
BOT_LIMIT_RE=''                                                       # posts no rate-limit notice
BOT_CLEAN_RE=''                                                       # a clean review still submits a review object
bot_rerequest() { gh pr edit "$1" --add-reviewer "@copilot"; }
# The failure notice is generic and can mask a HARD RATE LIMIT (weekly model cap, separate
# from the premium-request/credit budget -- which it doesn't even consult). Each review runs
# as Actions workflow "Copilot" recording the PR head at review time as its headSha, so
# --commit pins the current round's run; only that run's log names the real cause ("[SessionModelError]:
# You've reached your weekly rate limit. Please wait for your limit to reset on <date> ...",
# errorType rate_limit, HTTP 429) -- and the run concludes "success" even when the review
# failed (the error is reported in-session, not to the runner). Don't grep bare "429": log
# timestamps contain it. bot_status calls this before any retry. No match / unreadable log
# falls through to the transient retry path -- deliberate fail-open: the loop keeps moving
# and the 3-consecutive-failure cap bounds the waste.
bot_fail_diag() { gh run view "$(gh run list --workflow Copilot --commit "$(gh pr view "$1" --json headRefOid --jq .headRefOid)" --limit 1 --json databaseId --jq '.[0].databaseId')" --log 2>/dev/null | grep -m1 -oE "SessionModelError.{0,20}reached your [a-z]+ rate limit\.[^.]*\."; }

# --- CodeRabbit --- (auto-reviews on push -- incrementally, new changes only, and EVERY push
#     spends a review: batch a round's commits into one push, see One Push Per Round. Re-request
#     on demand via a PR comment. "@coderabbitai review" = incremental; "@coderabbitai full review"
#     = from-scratch on all files (after big rebases/refactors). Every trigger comment gets an
#     ACK within ~a minute -- read it before waiting on anything:
#       "Action performed / Review triggered."        -> accepted and running: poll, do not re-request
#       "Action not completed / Review rate limited." -> bounced, with a "Next review available
#         in: N minutes" window. A bounced request is NEVER queued: when the window reopens,
#         nothing runs until a fresh trigger is posted -- schedule the re-trigger at window
#         + 1-2 min slack, then read ITS ack too.
#     Quota-limited per plan (hourly buckets): instead of a review it may post "Review limit
#     reached ... Next review available in: N minutes" -- the quota refills over time. Auto-reviews
#     also silently pause after 5 reviewed commits by default (auto_pause_after_reviewed_commits)
#     -- resume with "@coderabbitai resume", not more review requests.)
BOT_REVIEW_LOGIN='coderabbitai[bot]'
BOT_THREAD_PREFIX='coderabbit'
BOT_FAIL_RE=''                                                        # no distinct failure-review phrase
# The quota notice is normally EDITED INTO the summary comment, where it carries an HTML marker
# ("<!-- This is an auto-generated comment: rate limited by coderabbit.ai -->") -- the most
# reliable match, since the visible heading varies ("Review limit reached" for the PR-review
# bucket, "Rate limit exceeded" for the commits/files bucket). "review rate limited" matches the
# bounced-trigger ack, which carries the same window. bot_status dates and honors all of them.
BOT_LIMIT_RE='rate limited by coderabbit\.ai|review limit reached|rate limit exceeded|review rate limited'
# A clean CodeRabbit review submits NO review object -- the only evidence is a comment: the
# walkthrough saying "No actionable comments were generated in the recent review", or -- after
# a redundant re-request -- the "Action performed / Review finished." ack, which means the
# incremental system already covered HEAD and no new review will ever come. Terminal "done",
# NOT a failure or rate limit. ("Actionable comments posted: 0" is the review-body variant of
# the same signal, kept as a hedge; asterisk-tolerant since the count is bolded.)
BOT_CLEAN_RE='no actionable comments were generated|actionable comments posted:[ *]*0|review finished'
bot_rerequest() { gh pr comment "$1" --body "@coderabbitai review"; }
```

Each bot uses a `[bot]`-suffixed login on REST and an unsuffixed one on GraphQL threads (Copilot: `copilot-pull-request-reviewer[bot]` / `copilot-pull-request-reviewer`; CodeRabbit: `coderabbitai[bot]` / `coderabbitai`). Copilot uses a third form -- plain `Copilot` -- on REST inline comments **and** in `requested_reviewers` (the surface `bot_requested` reads); the lowercase `BOT_THREAD_PREFIX` prefix match covers all three, so the two vars above suffice.

## bot_status (read-only detector)

The authoritative "done?" signal is **zero unresolved threads from this bot**, gated on the bot having reviewed the current HEAD. Order matters: **outstanding comments are checked first, PR-wide** -- if any exist, the verdict is "not clean" no matter which commit they were made on. Re-requesting over them would only produce a duplicate round (the auto-review collision); only the clean-verdict check is matched to HEAD. The HEAD-review evidence differs per bot: Copilot always submits a review object, but a clean CodeRabbit review submits **none** -- its evidence is a comment (`BOT_CLEAN_RE`): the summary comment's "no actionable comments" text (edited in place on re-reviews, so matched on `updated_at`, not `created_at`, and confirmed by the head sha in its commit range) or the "Review finished." ack.

**Exit `0` means "no unresolved threads", not "no findings"** -- CodeRabbit writes nitpicks and outside-diff findings into the review *body*, where they have no thread at all. Run the completeness check in Findings That Never Become Threads before treating `0` as done.

```bash
# Usage: bot_status <PR_NUMBER>   (read-only; a Per-Bot Setup block must be sourced)
# Exit: 0 clean (no unresolved threads; HEAD covered by a review object or, where BOT_CLEAN_RE
#       is set, by a clean-review notice) | 2 not clean | 3 pending
#       4 failed, no retry pending | 6 rate-limited (CodeRabbit: the newest notice's own
#       timestamp + its parsed window has not elapsed; Copilot: hard limit found in the review
#       run's CI log -- never retry)
# Uses gh's {owner}/{repo} placeholders + inline $(...) so each command matches the allowlist
# patterns on its own -- no owner=/repo=/head= assignments (a VAR= prefix breaks matching).
bot_status() {
  pr="$1"

  # 1) Unresolved bot threads, PR-wide: outstanding comments always win. Handle them before
  #    any re-request -- a new review on top of them just duplicates the points.
  threads="$(gh api graphql -f query="{ repository(owner: \"$(gh repo view --json owner --jq .owner.login)\", name: \"$(gh repo view --json name --jq .name)\") { pullRequest(number: $pr) { reviewThreads(first: 100) { totalCount nodes { isResolved path line comments(first: 1) { nodes { author { login } body } } } } } } }" --jq '.data.repository.pullRequest.reviewThreads')"
  unresolved="$(printf '%s' "$threads" | jq --arg p "$BOT_THREAD_PREFIX" '[.nodes[] | select(.isResolved==false and ((.comments.nodes[0].author.login // "") | ascii_downcase | startswith($p)))]')"
  n="$(printf '%s' "$unresolved" | jq 'length')"
  if [ "${n:-0}" -gt 0 ]; then
    echo "Unresolved $BOT_THREAD_PREFIX threads ($n):"
    printf '%s' "$unresolved" | jq -r '.[] | "  \(.path):\(.line)  \(.comments.nodes[0].body | gsub("\n";" ") | .[0:90])"'
    return 2
  fi
  # Zero unresolved is trustworthy only if we saw every thread: reviewThreads(first: 100) does
  # not paginate, and threads 101+ could hide outstanding comments (which must block a re-request).
  [ "$(printf '%s' "$threads" | jq '.totalCount')" -gt 100 ] && { echo "Inconclusive: >100 threads exceed the 100 fetched"; return 2; }

  # 2) No outstanding comments -- trust "clean" only from a review of the current HEAD.
  #    --paginate applies -q/--jq PER PAGE, so use --slurp (an array of pages) | jq and
  #    flatten with .[][] to pick the overall latest. Empty-body review objects are reply
  #    containers, not reviews: every inline reply (the bot's or yours) submits one at the
  #    current HEAD, so without the body-length filter a reply shell reads as "reviewed HEAD".
  review="$(gh api repos/{owner}/{repo}/pulls/$pr/reviews --paginate --slurp | jq -c --arg login "$BOT_REVIEW_LOGIN" --arg head "$(gh pr view "$pr" --json headRefOid --jq .headRefOid)" '[.[][] | select(.user.login==$login and .commit_id==$head and ((.body|length) > 0))] | last')"
  if [ -n "$review" ] && [ "$review" != "null" ]; then
    if [ -n "$BOT_FAIL_RE" ] && printf '%s' "$review" | jq -r '.body' | grep -iqE "$BOT_FAIL_RE"; then
      # The failure notice is generic -- consult the review run's CI log before treating it as
      # transient: a hard rate limit outlives every re-request (each still burning a billable
      # round + Actions minutes for a guaranteed failure).
      if type bot_fail_diag >/dev/null 2>&1; then
        lim="$(bot_fail_diag "$pr")" && [ -n "$lim" ] && { echo "$BOT_THREAD_PREFIX review FAILED: hard rate limit, not transient -- $lim -- never re-request before the reset; report it to the user"; return 6; }
      fi
      # A later re-request supersedes the failed review: while it is pending this is "waiting",
      # not "failed" -- and once the retry lands it becomes `last` and is judged instead.
      bot_requested "$pr" && { echo "$BOT_THREAD_PREFIX review FAILED on current HEAD -- retry already requested, waiting"; return 3; }
      echo "$BOT_THREAD_PREFIX review FAILED on current HEAD -- cool down ~5 min, then re-request"; return 4
    fi
    echo "No unresolved $BOT_THREAD_PREFIX threads -- clean."; return 0
  fi

  # 3) No review at HEAD. If the bot signals failures via a PR comment (BOT_FAIL_RE set), tell
  #    "failed" apart from "still pending". (A force-push that rewrites committer dates can delay
  #    this a tick.)
  if [ -n "$BOT_FAIL_RE" ]; then
    failed="$(gh api repos/{owner}/{repo}/issues/$pr/comments --paginate --slurp | jq --arg since "$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')" --arg p "$BOT_THREAD_PREFIX" --arg fail "$BOT_FAIL_RE" '[.[][] | select((.user.login|ascii_downcase|startswith($p)) and (.created_at > $since) and (.body|test($fail;"i")))] | length')"
    if [ "${failed:-0}" -gt 0 ]; then
      if type bot_fail_diag >/dev/null 2>&1; then
        lim="$(bot_fail_diag "$pr")" && [ -n "$lim" ] && { echo "$BOT_THREAD_PREFIX review FAILED: hard rate limit, not transient -- $lim -- never re-request before the reset; report it to the user"; return 6; }
      fi
      bot_requested "$pr" && { echo "$BOT_THREAD_PREFIX review FAILED (error notice) -- retry already requested, waiting"; return 3; }
      echo "$BOT_THREAD_PREFIX review FAILED (error notice in PR comments) -- cool down ~5 min, then re-request"; return 4
    fi
  fi

  # 4) Quota notice (BOT_LIMIT_RE set): instead of a review the bot answers with a limit notice.
  #    CodeRabbit normally EDITS it into the summary comment (HTML marker "rate limited by
  #    coderabbit.ai"); a bounced trigger posts it as an ack. Two window shapes exist, so the
  #    parse sums EVERY <number><h|m|s> pair in the phrase instead of grabbing one number:
  #      "**Next review available in:** **21 minutes**"    (PR-review bucket; bolded, and the
  #                                                         value can wrap onto the next line)
  #      "Please wait **13 minutes and 58 seconds** ..."   (commits/files-per-hour bucket)
  #    A minutes-only regex reads "1 hour and 5 minutes" as 5 minutes and re-triggers into a
  #    still-closed window; unparsable -> 30 min. Deadline = the notice's own timestamp
  #    (updated_at -- the edit that wrote it; created_at never moves on an edited comment) +
  #    window + 60s slack, and the newest match is picked by timestamp (max_by), not list
  #    position: the REST list is ID-ordered, which edits decouple from recency.
  #    NOT filtered by the last commit date -- the window is wall-clock, so a notice that
  #    predates your newest push still binds, and an expired one is harmless (the deadline
  #    decides). Staleness is judged instead by the head sha the notice names in its own commit
  #    range ("... between <base> and <head>."): the summary comment keeps that whole block
  #    through later edits while updated_at moves on, so a block naming an older head is a
  #    leftover, not a live window. The range head is read out of the text and compared
  #    prefix-wise in both directions, so an abbreviated sha on either side still matches.
  #    Notices with no commit range (the bounce ack) always count.
  lim_at=0; lim_until=""
  if [ -n "$BOT_LIMIT_RE" ]; then
    lim="$(gh api repos/{owner}/{repo}/issues/$pr/comments --paginate --slurp | jq -r --arg p "$BOT_THREAD_PREFIX" --arg lim "$BOT_LIMIT_RE" --arg head "$(gh pr view "$pr" --json headRefOid --jq .headRefOid)" '[.[][] | select((.user.login|ascii_downcase|startswith($p)) and (.body|test($lim;"i")) and (((.body|ascii_downcase|(capture("between +[0-9a-f]{7,40} +and +(?<h>[0-9a-f]{7,40})")? // {h:""})).h) as $h | $h == "" or ($head|startswith($h)) or ($h|startswith($head)))) | {t:((.updated_at // .created_at)|fromdateiso8601), w:((.body|ascii_downcase|(capture("(?:please wait|available in)[^0-9]{0,12}(?<d>[0-9][0-9a-z ,*]{0,40})";"i")? // {d:""})).d | [scan("([0-9]+)[ *]*([hms])")] | map((.[0]|tonumber) * (if .[1]=="h" then 3600 elif .[1]=="m" then 60 else 1 end)) | add // 1800)}] | max_by(.t) | if . == null then empty else "\(.t + .w + 60) \(.t)" end')"
    [ -n "$lim" ] && { lim_until="${lim%% *}"; lim_at="${lim##* }"; }
  fi

  # 5) Clean-review notice (BOT_CLEAN_RE set): no review object is coming -- a clean CodeRabbit
  #    review never creates one, so a comment is the only evidence, in two grades:
  #      names HEAD -- the summary comment's clean text WITH the current head sha in its commit
  #                    range: that review ran and covered HEAD. Terminal, and it outranks any
  #                    quota notice (a later bounced re-request cannot un-review HEAD).
  #      dated only -- the "Review finished." ack, which carries no sha: trusted when it
  #                    postdates both the last commit and the newest quota notice. Ordering
  #                    matters -- an ack from before a blocked round would otherwise mask the
  #                    round that never ran.
  #    Without this check a clean round looks pending forever and the tick re-requests a review
  #    that already happened.
  clean_head=0; clean_at=0
  if [ -n "$BOT_CLEAN_RE" ]; then
    clean="$(gh api repos/{owner}/{repo}/issues/$pr/comments --paginate --slurp | jq -r --arg p "$BOT_THREAD_PREFIX" --arg clean "$BOT_CLEAN_RE" --arg head "$(gh pr view "$pr" --json headRefOid --jq .headRefOid)" --arg since "$(gh pr view "$pr" --json commits --jq '.commits[-1].committedDate')" '[.[][] | select((.user.login|ascii_downcase|startswith($p)) and (.body|test($clean;"i")))] | "\([.[] | select(((.body|ascii_downcase|(capture("between +[0-9a-f]{7,40} +and +(?<h>[0-9a-f]{7,40})")? // {h:""})).h) as $h | $h != "" and (($head|startswith($h)) or ($h|startswith($head))))] | length) \([.[] | select((.updated_at // .created_at) > $since) | ((.updated_at // .created_at)|fromdateiso8601)] | max // 0)"')"
    clean_head="${clean%% *}"; clean_at="${clean##* }"
  fi
  [ "${clean_head:-0}" -gt 0 ] && { echo "No unresolved $BOT_THREAD_PREFIX threads -- clean (clean-review notice names HEAD; no review object expected)."; return 0; }
  if [ -n "$lim_until" ] && [ "$(date +%s)" -lt "${lim_until%.*}" ]; then
    echo "$BOT_THREAD_PREFIX rate-limited -- next review available in ~$(( (${lim_until%.*} - $(date +%s)) / 60 + 1 )) min"
    return 6
  fi
  [ "${clean_at:-0}" -gt "${lim_at:-0}" ] && { echo "No unresolved $BOT_THREAD_PREFIX threads -- clean (trigger ack covers HEAD; no review object expected)."; return 0; }
  if bot_requested "$pr"; then
    echo "$BOT_THREAD_PREFIX review already requested (auto-review or an earlier request) -- waiting, no re-request needed"
  else
    echo "No $BOT_THREAD_PREFIX review yet for current HEAD (not requested)"
  fi
  return 3
}
```

## bot_requested (pending-request check)

Auto-review (and any earlier request) leaves the bot as a pending requested reviewer until its review lands. Re-requesting on top of that is the collision to avoid. **The pending entry is only visible on the REST endpoint** -- `gh pr view --json reviewRequests` omits bot reviewers entirely (verified: it stays `[]` while REST shows the request), so never use it for this check:

```bash
# Usage: bot_requested <PR_NUMBER>   (read-only; exit 0 = a request for this bot is pending)
bot_requested() { [ "$(gh api repos/{owner}/{repo}/pulls/$1/requested_reviewers --jq "[.users[].login | ascii_downcase] | any(startswith(\"$BOT_THREAD_PREFIX\"))")" = "true" ]; }
```

(Copilot appears there as login `Copilot` -- yet another identity form; the lowercase prefix match covers it. CodeRabbit is comment-triggered and never appears as a requested reviewer -- for it this check is simply always false, which is correct.)

## bot_tick (re-request only if needed + one bounded poll)

If there's already an outcome, return it; if a request is already pending (auto-review, or a previous tick's request), skip straight to polling; otherwise re-request once and poll until the review lands or a 5-minute deadline. On a **failed** review (Copilot's "encountered an error ... re-requesting a review" -- transient once `bot_status` has ruled out a hard rate limit via `bot_fail_diag`, and re-requesting then usually succeeds) it cools down 5 minutes for safety, issues the single re-request (skipped if one appeared during the cooldown), and returns `4` so the caller can count consecutive failures; the *next* tick polls the retry (`bot_status` reports `3` while it's pending). **At most one re-request, one bounded poll, no internal retry loop** -- so a single invocation stays well under the 10-minute Bash cap; the agent's loop owns retries and escalation.

```bash
# Usage: bot_tick <PR_NUMBER>   (needs the config block + bot_rerequest + bot_requested)
# Exit: 0 clean | 2 not clean | 3 pending/timed out | 5 unavailable (non-GitHub or re-request failed)
#       4 failed (already cooled down ~5 min + re-requested -- re-run to poll) | 6 rate-limited
bot_tick() {
  pr="$1"
  case "$(git remote get-url origin 2>/dev/null)" in
    *github.com*) : ;;
    *) echo "GitHub-only: origin is not github.com"; return 5 ;;
  esac
  bot_status "$pr"; rc=$?
  if [ "$rc" -eq 4 ]; then
    # Transient bot failure: cool down 5 min so the fault clears, then one re-request. The next
    # bot_tick polls it -- returning 4 here lets the caller count failures and cap the retries.
    sleep 300
    # A request may have appeared during the cooldown (a human, draft->ready) -- never double-request.
    if ! bot_requested "$pr" && ! bot_rerequest "$pr" >/dev/null 2>&1; then
      echo "Re-request after failed review did not go through"; return 5
    fi
    echo "Retry requested after failed review -- re-run bot_tick to poll it"; return 4
  fi
  [ "$rc" -ne 3 ] && return "$rc"            # already have an outcome (0 / 2 / 6)

  # Re-request only when no request is pending -- auto-review (non-draft PR creation,
  # draft->ready) or an earlier tick may already have one in flight.
  if ! bot_requested "$pr"; then
    if ! bot_rerequest "$pr" >/dev/null 2>&1; then
      echo "Re-request failed -- bot not enabled here, or wrong command (see Per-Bot Setup)"; return 5
    fi
  fi
  deadline=$(( $(date +%s) + 300 ))          # 5 min; one invocation stays well under the 10-min Bash cap
  while [ "$(date +%s)" -lt "$deadline" ]; do
    sleep 30; bot_status "$pr"; rc=$?
    [ "$rc" -ne 3 ] && return "$rc"
  done
  echo "No review within this tick -- re-run bot_tick (safe: it re-checks before re-requesting)"; return 3
}
```

## Findings That Never Become Threads (CodeRabbit)

**Only the findings CodeRabbit counts as *actionable* are posted inline as review threads.** The rest are written into the **review body**: no thread, no resolve button, and nothing in `reviewThreads` -- so a thread-only workflow silently drops them and the user has to point them out. This is the single most common "you missed comments" failure.

| Body-only bucket | Why it isn't inline |
|-----------------------------------|----------------------------------------------------------------------|
| `Nitpick comments (N)`            | Folded into the body by `profile: chill` (and `quiet`) to keep the diff clean |
| `Outside diff range comments (N)` | "Some comments are outside the diff and can't be posted inline due to platform limitations" |
| `Duplicate comments (N)`          | Repeats of an earlier finding on the same code -- handle the original thread |
| `Comments failed to post (N)`     | Inline posting failed; the finding exists only here                   |

Each is a collapsed `<details>` section whose summary is icon-prefixed in the rendered comment -- match the words, not the icon. One read-only call lists what every review carried:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --paginate --slurp | jq -r --arg p coderabbit '.[][] | select((.user.login|ascii_downcase|startswith($p)) and ((.body|length)>0)) | (.body|ascii_downcase) as $b | "\(.submitted_at) \(.commit_id[0:7]) inline=\(($b|capture("actionable comments posted:[ *]*(?<n>[0-9]+)")? // {n:"?"}).n) body-only=[\([$b|scan("(nitpick comments|outside diff range comments|duplicate comments|comments failed to post) *\\(([0-9]+)\\)")|join(" ")]|join("; "))]"'
```

For any non-zero bucket, read that review's full body and triage each item exactly like a thread -- fix it or reject it with a reason. There is nothing to resolve afterwards (no thread exists), so **record the verdicts** in the reply on a related thread or in one PR comment; otherwise the only trace is your commit and the user cannot tell them from work you skipped.

**Reconcile before declaring done.** The inline set is complete only when the count the bot claims matches the threads that exist:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate --slurp | jq --arg p coderabbit '[.[][] | select((.user.login|ascii_downcase|startswith($p)) and (.in_reply_to_id == null))] | length'
```

That number must equal the sum of `Actionable comments posted: N` across the bot's reviews (the `in_reply_to_id` filter drops the bot's replies, which are not findings). Fewer threads than claimed means comments failed to post -- the missing ones are in the review body.

## One Push Per Round

**On an auto-review repo every push IS a review request.** CodeRabbit re-reviews each push incrementally, and each of those reviews spends a slot from the hourly per-developer bucket exactly like a manual `@coderabbitai review`. Three commits pushed one at a time are three reviews of a change that was never finished; the same three commits pushed together are one review of the finished change.

1. **Commit as often as the change needs; push once.** Keep commits separate and conventional (`git-commit` skill) -- granularity is free. It is `git push` that spends the quota.
2. **Finish the round before pushing.** Fix every valid thread *and* every body-only finding, commit them all, then push -- never push a partial round and keep working.
3. **Expect the silent pause regardless.** `auto_pause_after_reviewed_commits` (default 5) counts *reviewed commits*, so batching pushes saves review slots but does not postpone the cap -- a long PR eventually stops being auto-reviewed, which reads as a broken bot. That state needs `@coderabbitai resume`, not more review requests.
4. **Iterate in draft when the change isn't ready.** Drafts are excluded from auto-review, so pushes to a draft cost nothing; marking ready is the request.
5. **A wasted push costs more than a wasted trigger.** A bounced trigger consumes nothing (it is never queued); a push that lands mid-round consumes a real review of incomplete work -- and the next round then has to re-review the same files.

## The Loop

The agent drives rounds; `bot_tick` is one mechanical round. **Validate every comment** -- bots are useful but can be out of context or working from outdated knowledge, so judge each on its merits and never blind-fix.

```text
INPUT: PR N; a Per-Bot Setup block sourced; MAX_ROUNDS = integer from the request ("loop 3" -> 3),
       else from the Code Review Policy, else 5
GATE:  billable review requests require an explicit loop instruction or a permissive policy --
       otherwise ASK before ANY tick that would issue one: after each processed round, and on
       round 1 when nothing is pending or reviewed yet (repos without auto-review; see intro)
prev_head = ""; round = 0; fails = 0; processed = 0
repeat:
  round += 1;  if round > MAX_ROUNDS: STOP "hit round cap -- escalate"
  if no explicit loop instruction and no permissive policy:
    if processed >= 1 or (no pending bot request and no bot review or clean-review notice at HEAD):
      # this tick would issue a billable request; auto-review rounds arrive without one
      # (read-only preflight: bot_requested + the review-at-HEAD and BOT_CLEAN_RE checks in bot_status)
      ASK "next review request bills fully -- proceed?" (recommend from the last round's finding quality)
      on no answer / decline: STOP "awaiting approval for the billable review request"
  head = gh pr view N --json headRefOid --jq .headRefOid
  if round > 1 and head == prev_head: STOP "no code change -- re-review would resurface the same points"
  prev_head = head
  bot_tick N:
    0 -> no unresolved threads. Before stopping, run the completeness check (Findings That Never
         Become Threads): triage any body-only buckets in this bot's reviews, and reconcile the
         claimed actionable count against the threads that exist. Only when both come out empty:
         STOP "clean -- this bot has no unresolved comments"
    5 -> STOP "not applicable -- non-GitHub remote, or the bot isn't enabled here"
    3 -> still pending: re-run bot_tick within the same round -- do NOT re-enter the round header,
         so the unchanged-HEAD guard never aborts a wait (it applies only after a processed review
         outcome); after a couple of timeouts STOP "timed out / maybe unavailable"
    4 -> transient failure (e.g. "Copilot encountered an error"). bot_tick already cooled down ~5 min
         and re-requested; fails += 1. If fails >= 3: STOP "keeps failing -- likely structural:
         oversized PR, binary/minified files, quota; fix the cause". Else re-run bot_tick (polls the
         retry). Retries don't consume a round -- no review happened.
    6 -> rate-limited. Copilot (hard weekly limit, diagnosed from the review run's CI log):
         STOP immediately and report the real cause + reset date from the log to the user --
         the reset is days away, and every re-request until then fails identically while
         still burning a billable round + Actions minutes. CodeRabbit (hourly window): if
         another review bot is active on this repo, STOP this bot's loop and rely on that
         one; otherwise wait out the printed window plus 1-2 min slack ("next review
         available in ~N min"; sleep in a background Bash), then re-run bot_tick -- the
         bounced request was never queued, so the re-run's fresh trigger is what starts
         the review (read its ack). Waits don't consume a round.
    2 -> not clean; fails = 0; processed += 1. If bot_status flagged the rare >100-thread inconclusive case,
         paginate/resolve to confirm before trusting clean. Otherwise validate each unresolved
         comment (pr-comment-workflow.md) AND the body-only buckets of the reviews behind them
         (Findings That Never Become Threads -- nitpick, outside-diff-range, duplicate and
         failed-to-post findings have no thread and are invisible to the query above):
           valid   -> fix
           invalid -> reply with the rationale + resolve (no code change)
         if NONE were valid (nothing to fix): STOP "zero valid comments -- re-requesting would only resurface them"
         else: commit every fix of this round -- as many commits as the change needs -- then ONE
               push for the whole round (One Push Per Round: each push is another billed
               incremental review; advances HEAD; message = the change itself, never the
               bot/round), then continue: push-triggered bots (CodeRabbit) re-review on their own unless
               auto-paused (silent after 5 reviewed commits -- see CodeRabbit specifics);
               Copilot re-reviews on push only if the ruleset sets review_on_push (off by
               default). Either way the next bot_tick covers it --
               it re-requests only when no review at HEAD and nothing pending (no duplicate
               requests; an explicit @coderabbitai review works even while auto-paused)
```

**It always terminates.** A round continues only when a *valid* comment was fixed (advancing HEAD); an all-rejected round stops at the zero-valid exit (the `head == prev_head` guard is a backstop). Transient failures are retried with a cooldown + re-request but stop after 3 consecutive failed reviews, rate limits wait a bounded window or defer to another bot, unavailability stops immediately, and `MAX_ROUNDS` caps the whole thing -- so it converges on substance.

To clear *every* reviewer in one pass, run the loop once per active bot (and address human threads via `pr-comment-workflow.md`); each bot's `bot_status` only counts its own threads.

## Scheduling Several PRs Through One Bucket (CodeRabbit)

The PR-side quota is **per developer, not per PR** (docs: limits are enforced per developer over rolling hourly windows): one bucket spans every PR in every repo, so all of a developer's open PRs contend for the same review windows -- including PRs being merged in parallel, whose auto-reviews consume slots just like re-requests. The bucket is a rolling hourly allowance (Free: 1 PR review/hour, Pro: 5, Pro+: 10 -- full table in the `coderabbit` skill); trials and adaptive throttling can shrink the effective rate to roughly **one review per ~25-30 minutes**, at which point several PRs in flight is a scheduling problem, not N independent loops:

1. **Drafts are the queue control.** Draft PRs are excluded from auto-review by default (`auto_review.drafts: false`) and cost nothing until manually triggered; with auto-review enabled, **marking one ready IS the auto-review request** (auto-review off or gated: post `@coderabbitai review` after promoting) -- promote one PR at a time, when it is that PR's turn for the window.
2. **One window, one PR, one trigger.** Give the next window to the PR closest to merge. Parallel triggers race for the remaining allowance -- on a depleted bucket, a single slot -- and the losers bounce with the rate-limit ack; a bounced request is never queued (Per-Bot Setup), so it must be re-posted in a later window.
3. **Prep waiting drafts with the CLI lane meanwhile.** Local CodeRabbit CLI reviews (`coderabbit` skill) draw from a separate hourly bucket (included allowances only -- usage-based overage credits are one shared org pool), so a draft arrives at its PR-side round already CLI-clean and typically needs only one confirming window.
4. **Merge as soon as clean.** Bot-clean is not the merge bar by itself: the repo's merge requirements (CI, required human approvals -- bot reviews satisfy none of them) still apply. Once they pass, merge immediately -- a merged PR stops contending and frees the next window for the queue.

## How Bot Reviews Appear

- **State is always `COMMENTED`** -- these bots never approve, request changes, block a merge, or satisfy a required-approval rule. Don't use `reviewDecision` to detect them.
- **Identity differs by surface and by bot** -- a `[bot]`-suffixed login on REST, unsuffixed on GraphQL threads (see Per-Bot Setup). Filter the wrong one and detection silently returns nothing.
- **A pending request is only visible via REST** -- while requested (auto-review or manual), Copilot appears as login `Copilot` in `gh api repos/{owner}/{repo}/pulls/{n}/requested_reviewers`; the entry disappears when the review is submitted. `gh pr view --json reviewRequests` omits bot reviewers entirely (it shows humans/teams only), so it always looks "not requested" -- the wrong surface. That pending window is when a manual re-request collides -- `bot_requested` guards it.
- **Copilot specifics** -- requires gh >= 2.88; does **not** auto re-review on push unless the ruleset's `review_on_push` is set (otherwise re-request every round; repo auto-review covers only round 1); **every review bills fully** -- 13 premium requests (legacy annual plans) or AI credits + Actions minutes (current plans; see `copilot-review-config.md`), with no re-request discount. An exhausted quota **and a hard weekly rate limit** both fail the review with the same opaque error comment as a transient failure: `Copilot encountered an error and was unable to review this pull request. You can try again by re-requesting a review.` (detect via `BOT_FAIL_RE`, never treat as clean). The comment is misleading -- the review run's **Actions log is the ground truth**: each review runs as workflow `Copilot` (event `dynamic`, job `copilot-pull-request-reviewer`), so `bot_fail_diag` greps the latest run's log on the head branch. A `SessionModelError ... You've reached your weekly rate limit. Please wait for your limit to reset on <date> ...` (errorType `rate_limit`, HTTP 429) is a hard model cap separate from the premium-request/credit budget -- never re-request before the logged reset; report the date instead. With no such log line the failure is usually **transient**, and a re-request after a short cooldown typically succeeds (`bot_tick` automates the ~5-min wait + re-request); when retries keep failing near end-of-month, check the quota before blaming PR size. The summary says `... generated K comments` (singular at `K==1`).
- **CodeRabbit specifics** -- auto-reviews each push when auto-review is enabled (the default; **incremental**: new changes only), so a push usually re-triggers it; re-request on demand with a `@coderabbitai review` PR comment (also incremental), or `@coderabbitai full review` for a from-scratch pass over all files (after big rebases/refactors, or when early reviews predate significant context). Every trigger comment gets an **ack within ~a minute** -- read it instead of blind-polling: "Action performed / Review triggered." means accepted and running (poll, don't re-request); "Action not completed / Review rate limited." means bounced with the `Next review available in: N minutes` window, and **never queued** -- after the window, only a fresh trigger starts a review. Auto-reviews **silently pause after 5 reviewed commits** by default (`auto_pause_after_reviewed_commits`) -- a long-running PR that "stopped getting reviews" needs `@coderabbitai resume`, not more requests. Resolve its threads like any other (or use its `@coderabbitai resolve` command). **A clean review is invisible on the reviews API**: with zero actionable comments CodeRabbit submits no review object -- the walkthrough comment ("No actionable comments were generated in the recent review" / "Actionable comments posted: 0", edited in place on later reviews) is the only evidence, and re-requesting anyway just gets the "Action performed / Review finished." ack: the incremental system has already covered these commits and no new review will come. That ack is a terminal "done" -- never read it as a failure or rate limit (`bot_status` detects all of this via `BOT_CLEAN_RE`). **Not every finding is a thread**: with `profile: chill` the nitpicks, plus any outside-diff-range or failed-to-post findings, live only in the review **body** (Findings That Never Become Threads) -- `Actionable comments posted: N` counts the inline ones alone. **All plans are quota-limited** (refilling per-hour review buckets; Free: 1 PR review/hour, summary only; Pro tiers add adaptive limits under sustained volume), and **each push consumes a review** just like a manual trigger, so batch a round's commits into one push (One Push Per Round). When the bucket is empty it posts `Review limit reached` / `Rate limit exceeded` with the window as either `Next review available in: N minutes` or `Please wait N minutes and M seconds`, usually **edited into the existing summary comment** rather than posted fresh, and that comment keeps the block through later edits (detect via `BOT_LIMIT_RE`; `bot_status` dates it by `updated_at`, sums h/m/s, checks the head sha in the block's commit range, and returns `6` only while the window is still binding). Re-requesting inside the window is pointless; wait it out -- or, when Copilot is also active on the repo, just rely on Copilot and skip CodeRabbit for that round. The buckets are **per developer, not per PR** -- all of a developer's open PRs across repos share one pool; see Scheduling Several PRs Through One Bucket. `.coderabbit.yaml` tuning and the local CLI flow live in the `coderabbit` skill.

## Preconditions

- **GitHub only** -- these are GitHub review bots; `bot_tick` checks the remote and exits `5`.
- **The bot must be enabled** -- Copilot code review needs a paid plan that includes it plus org/enterprise + repo settings (Settings > Copilot > Code review); CodeRabbit needs its GitHub App installed on the repo. Detection is partial: the Copilot auto-review **ruleset** is readable (`copilot_code_review` rule via `gh api repos/{owner}/{repo}/rules/branches/{branch}`; 403 on Free-plan private repos = unknown), but neither bot's overall availability nor the personal auto-review setting has a read-only endpoint -- a re-request that errors (exit `5`) is the signal; if requests are accepted but no review arrives, that's exit `3` (slow *or* silently unavailable). Conversely a bot may be **auto-requested** (ruleset or personal setting) -- a pending REST `requested_reviewers` entry or an unrequested bot review on a fresh PR is the tell; never issue a redundant request there.
- **Allowlist** -- the read-only polls and the opt-in re-request have patterns in `allowlist.md`. Keep REST paths unquoted (`gh api repos/...`) and issue the re-request in the canonical form, or matching fails and it prompts.

## Autonomy & allowlisting

For unattended loops the commands must match the patterns in `allowlist.md`:

- **Read-only detection + the opt-in re-request are allowlist-matchable.** The driver uses `gh api repos/{owner}/{repo}/...` placeholders and inline `$(...)` -- never `owner=`/`repo=`/`head=` assignments (a `VAR=` prefix stops matching) -- and keeps REST paths unquoted, so each command auto-approves under the documented patterns.
- **The helper functions are one shell invocation.** A permission check applies to the whole `bot_status`/`bot_tick` call, not each inner command, so in prompt-mode you approve once per call. For a fully hands-off loop, run in an auto-approve permission mode (or invoke the individual commands, which match the patterns).
- **The per-round writes are the deliberate gate.** Addressing comments -- reply, resolve thread, `git commit`/`push` -- is not auto-approved by default; that's the human checkpoint. Allow those too (or use an auto-approve mode) for end-to-end autonomy.

## Key Gotchas

1. **Identity differs by surface and bot** (Per-Bot Setup) -- the most common detection bug; filter the right login.
2. **A failed review looks like "no comments"** (Copilot) -- detect the error phrase; never treat it as clean. The generic comment masks three distinct causes; the review run's CI log tells them apart (`bot_fail_diag`, run automatically by `bot_status` before any retry). A logged `rate_limit`/429 `SessionModelError` is a **hard weekly limit**: never re-request before the reset date the log states -- each attempt fails identically while still burning a round + Actions minutes; report the date to the user. No log hit -- usually transient: cool down ~5 min, re-request (`bot_tick` does both), escalate only after repeated failures. **Exhausted quota also fails with the identical error**: on repeated failures, check billing usage (`copilot-review-config.md`) before more retries.
3. **A rate-limited bot looks stuck, not broken -- and the window is the part that gets computed wrong** (CodeRabbit, any plan). The notice states when the quota refills in one of two shapes (`Next review available in: N minutes`, bolded and sometimes wrapped onto the next line; `Please wait N minutes and M seconds`), so **parse hours, minutes and seconds and sum them** -- a minutes-only regex turns "1 hour and 5 minutes" into 5 minutes and re-triggers into a window that is still closed. It normally arrives as an **edit to the summary comment** (`created_at` never moves -- date the window by `updated_at`), and that comment **keeps the whole block through later edits**, so check the head sha in the block's own commit range to tell a live window from a leftover. Do **not** gate the check on the last commit date: the window is wall-clock and still binds after a new push, and an expired notice is harmless because the deadline decides. Then wait it out and post a **fresh** trigger -- a bounced request is never queued, so nothing runs at window-reopen on its own. Or prefer Copilot when both bots are installed.
4. **Stop on zero *valid* comments, not zero comments** -- reject out-of-context/outdated points with a rationale + resolve; don't blind-fix to silence a bot, and don't chase one that resurfaces rejected points.
5. **Never request a review over outstanding work** -- unresolved bot comments mean *handle them*; a pending request (auto-review) means *wait*. `bot_status` (threads first) and `bot_requested` encode both; requesting anyway yields a duplicate round of the same points.
6. **Pending bot requests hide from `gh pr view --json reviewRequests`** -- that field omits bot reviewers, so it reads `[]` while a Copilot request is in flight. Only REST `requested_reviewers` (login `Copilot`) tells the truth; checking the wrong surface re-introduces the duplicate-request collision.
7. **Each round is billed -- treat re-requests as spending, not retrying** -- a Copilot re-request costs the same as the first review (13 premium requests legacy / credits + Actions minutes); without an explicit loop instruction or permissive policy, ask after the first processed round instead of looping. Iterate locally first (CodeRabbit CLI, `coderabbit` skill) so PR rounds only confirm.
8. **A CodeRabbit PR that "stopped being reviewed" is usually paused, not broken** -- the default `auto_pause_after_reviewed_commits: 5` halts auto-reviews on long PRs; `@coderabbitai resume` restarts them, while extra review requests just burn quota.
9. **A clean CodeRabbit review looks like no review at all** -- zero actionable comments means **no review object** on `/pulls/{n}/reviews`; the clean walkthrough comment is the only evidence (edited in place on re-reviews, so match `updated_at`, not `created_at`), and the "Review finished." ack after a re-request means HEAD is already covered -- terminal "done", not a failure or rate limit. Without the `BOT_CLEAN_RE` check the loop waits forever, then re-requests a review that already happened and misreads the ack.
10. **The CodeRabbit PR bucket is per developer, not per PR -- schedule PRs, don't race them** -- all of a developer's open PRs contend for the same windows, and parallel triggers race for the remaining allowance (the losers bounce, unqueued). Keep waiting PRs as drafts (excluded from auto-review; marking ready is the request), give each window to the PR closest to merge, and prep drafts with the CLI lane meanwhile (separate bucket) -- see Scheduling Several PRs Through One Bucket.
11. **A review object at HEAD is not proof the bot reviewed HEAD** -- every inline reply (the bot's or yours) is wrapped in an empty-body `COMMENTED` review submitted at the current HEAD, so a reply-container shell can sit at a commit the bot never reviewed (e.g. while its incremental re-review bounced on quota). Filter empty bodies when keying "clean" off a HEAD review (`bot_status` does).
12. **Zero unresolved threads is not zero findings** -- CodeRabbit puts nitpicks (`profile: chill`), outside-diff-range findings and failed-to-post comments in the review **body**, where `reviewThreads` never sees them; a PR can read "clean" with four untouched findings in it. Triage the body buckets and reconcile the claimed `Actionable comments posted: N` against the threads that exist before calling a round done (Findings That Never Become Threads).
13. **One push per round -- pushes are what spend the quota** -- on an auto-review repo each push triggers another incremental review from the same per-developer hourly bucket, so three commits pushed separately are three reviews of a change that was never finished. Commit as often as the change needs, but push once, after every fix in the round is committed; iterate in draft (not auto-reviewed) while the change is still moving.
