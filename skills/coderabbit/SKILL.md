---
name: coderabbit
description: >-
  CodeRabbit CLI local code reviews and .coderabbit.yaml configuration.
  Run AI reviews on uncommitted/committed changes before pushing or opening
  a PR, parse --agent findings, iterate fix-and-verify loops within hourly
  rate limits, and tune repo config for low-noise high-signal reviews. Also
  drives cloud Coding Agent tasks from the CLI (coderabbit code). Use when
  reviewing local changes pre-push/pre-PR, installing or driving the
  coderabbit/cr CLI, starting or following a CodeRabbit cloud task, creating
  or tuning .coderabbit.yaml, or reducing CodeRabbit review noise. Not for
  PR-side review loops and thread handling (git-pr), commits (git-commit),
  or CI status (git-ci)
---

# CodeRabbit CLI and Configuration

**Local-first AI code review: catch issues before they reach the PR.** The CodeRabbit CLI (`coderabbit`, alias `cr`) reviews working-tree or branch changes locally, so the PR-side review becomes confirmation instead of iteration -- this saves billable PR review rounds (both CodeRabbit's own quota and any Copilot credits). Covers the CLI surface -- local reviews, plus the beta cloud Coding Agent commands -- and `.coderabbit.yaml` tuning. PR-side mechanics (threads, re-requests, the bot loop) live in the `git-pr` skill.

**Verified against CodeRabbit CLI v0.9.1** (2026-10-22): every subcommand's `--help` on the official build, hidden subcommands included, diffed against v0.9.0, and the hidden and removed review flags run on a clean tree; plus the [CLI changelog](https://docs.coderabbit.ai/changelog), which has no v0.9.1 entry yet, and the [agent-mode](https://docs.coderabbit.ai/cli/agent-mode), [cloud-task](https://docs.coderabbit.ai/cli/cloud-tasks) and [review-command](https://docs.coderabbit.ai/reference/review-commands) pages. Flags are tagged with the version that introduced them. The billed and pushing `coderabbit code` commands were not exercised; their handling comes from those docs, and for the v0.9.1 additions the docs do not cover yet (`code archive`, `code goal`, `new --goal`/`--private`/`--access`, `resume --interactive`) from `--help` alone. If `coderabbit --version` reports something newer, diff `--help` of each subcommand against this list and read the changelog entries since v0.9.1 before relying on flag details here.

## When to Use

- **Reviewing local changes** -- "review my changes", "run coderabbit", pre-commit/pre-push/pre-PR checks
- **Driving a review-fix loop** -- run review, fix valid findings, re-run to verify
- **Configuring CodeRabbit** -- create or tune `.coderabbit.yaml`, reduce review noise, disable redundant linters
- **Checking limits** -- rate limits per plan, review budgeting
- **Setting up the CLI** -- install, auth, updates, headless/CI usage
- **Running cloud Coding Agent tasks** -- `coderabbit code new`/`resume`/`push`, handing a local session off to the cloud (beta, v0.9.0)

## Critical Rules

1. **Reviews upload code to CodeRabbit's service.** A review sends the diff (and context) to CodeRabbit. On a repo with sensitive/unpublished code, confirm the user is OK with that before the first run.
2. **Reviews consume a per-hour quota** (Free: 3/hour CLI reviews). Scope deliberately (`--committed`/`--uncommitted`, `--base`) and use `coderabbit review findings` to replay the last result without spending a review.
3. **Use `--agent` output when driving fixes programmatically**; the default plain-text mode is for humans. **Never pass `--plain`**: no `--help` lists it, and what it does depends on the binary. v0.7 removed it -- `error: unknown option` plus the usage text, yet exit `0`, so a piped or tailed run looked clean while nothing was reviewed; v0.8 and v0.9.0 reject it with exit `1` (v0.9.0 adds a `Remove --plain` hint); v0.9.1 accepts it again as a hidden no-op. Do not guess flags: `coderabbit review --help` on the installed binary is the supported surface, and the hidden legacy aliases it omits still parse -- `--type <all|committed|uncommitted>` and `--light` both start a real review, and v0.9.1 takes the removed `--prompt-only` as `--agent`.
4. **Validate findings before fixing** -- same rule as PR reviews: judge each finding on its merits; never blind-fix to silence the tool.
5. **CodeRabbit is optional -- check the Code Review Policy first** (repo AGENTS.md/CLAUDE.md, falling back to the user's global agent instructions; see `git-pr` skill) for the preferred reviewer and checkpoints. Never install, authenticate, or run it on a project whose policy or user hasn't opted in.
6. **`coderabbit code` spends money and pushes code.** `new`, `ask`, and anything sent through `resume` (`-m`, `--steer`, `--answer`, or an `--interactive` session) start billed agent work; `push` and Autofix (formerly Autopilot) write to the repository, and editing a completed `code goal` restarts the work. Run them only on the user's explicit request, and never rerun one that already reported its commit point -- the rerun duplicates the billed work (`references/cloud-tasks.md`).

---

## Setup

```bash
curl -fsSL https://cli.coderabbit.ai/install.sh | sh   # also links `cr`; or: brew install coderabbit (no `cr`)
coderabbit auth login        # browser OAuth; --agent emits JSON for agent-driven login
coderabbit auth status       # verify
coderabbit doctor            # diagnose runtime/auth/git/connectivity; exits 1 when any check fails
```

`auth login` also accepts `--api-key "<key>"` (store a key instead of OAuth), `--region us|eu`, `--self-hosted`, and `--no-browser` (print the URL for a browser that can reach the localhost callback). `auth org` switches the active organization interactively; since v0.9.0 `auth org --agent` only **lists** the organizations as JSON, so an agent cannot switch -- a person runs `coderabbit auth org`. Every `auth` subcommand takes `--agent` for JSON output.

**No paid plan required**: the Free plan includes CLI reviews (3/hour) after `coderabbit auth login` with a free account -- paid plans add org context, learnings, and higher limits. Headless/CI: an **Agentic API key** (a user API key is rejected) from an organization where that user holds a seat, passed per run as `--api-key "$CODERABBIT_API_KEY"` or stored once with `coderabbit auth login --api-key "$CODERABBIT_API_KEY"`. **The CLI reads no API-key environment variable** -- `CODERABBIT_API_KEY` is only the conventional secret name. Reviews draw on that user's plan allowance first, and on usage credits only where the usage-based add-on is on.

**Updating**: `coderabbit update` updates an install-script install; **a Homebrew install updates only with `brew upgrade coderabbit`** (v0.9.0 `update --help`; `doctor` reports `Update policy: Managed by Homebrew`). Outside Homebrew the CLI also updates itself: `CODERABBIT_CLI_DISABLE_AUTO_UPDATE=true` holds the installed version, and `CODERABBIT_VERSION=X.Y.Z` on the install script pins one. Other switches listed in v0.9.0 `--help`: `CR_CLI_TELEMETRY_DISABLED=true`, and `CODERABBIT_LOG_LEVEL=debug` with `CODERABBIT_LOG_TRANSPORT=console` to send logs to stderr instead of `~/.coderabbit/logs`. `coderabbit completion bash|zsh|fish|powershell` (v0.9.0) prints a completion script for both names.

Local reviews must run inside a git worktree (`--dir` can target one); `--remote` reviews and `pullrequest <full URL>` work from anywhere. **`cr` is a symlink the install script creates (`~/.local/bin/cr`); the Homebrew cask links only `coderabbit`** -- write commands, scripts, and allowlist patterns with the full name.

## Review Scopes and Checkpoints

| Checkpoint | Command | Reviews |
|------------|---------|---------|
| Before commit | `coderabbit review --uncommitted` | Staged + unstaged tracked changes |
| Before push | `coderabbit review --committed` | Local commits not in the base branch |
| Before PR (full branch) | `coderabbit review --base main` | Working tree + branch commits vs base |
| Subdirectory only | `coderabbit review --dir packages/api` | Changes under a path |
| Vs specific commit | `coderabbit review --committed --base-commit {sha}` | Changes since a commit |
| Include untracked files | `coderabbit review --include-untracked` | Also files not yet added to git |

Defaults (CLI v0.8+): all tracked changes, base = repository default branch, plain-text output, the local review policy. `--include-untracked` combines with `--uncommitted` but not with `--committed`, and contradictory scopes are rejected before a review starts. v0.7 **removed** `--plain` (`error: unknown option` through v0.9.0, a silent no-op again in v0.9.1) -- plain is simply the default. `--type <all|committed|uncommitted>` was never removed: `--help` hides it, it still runs a review, and it errors next to `--committed`/`--uncommitted` (`cannot be used with option '-t, --type <type>'`) -- scope with the listed flags instead.

**Local reviews keep a checkpoint (v0.7.7+).** Each review saves context for its scope -- review directory, branch, and base -- that later runs in the same scope reuse; a `--dir src` run and a whole-repository run keep separate checkpoints, changing the base resets it, and a failed review keeps the last successful one. The docs call this incremental, but it does not narrow what gets reported: observed on this repo (2026-09-30), a second `--committed --base main` run on the same branch reported six findings in commits the first run had passed with zero. Treat every run as a review of the whole scope, and a clean pass as one sample, not proof. `--fresh` (v0.8.2) discards the checkpoint and reviews the scope without the saved context.

**Review depth (v0.8)**: `--deep` runs the **full pull request review policy** locally -- the closest the CLI gets to what the PR-side review will say. It stays under the CLI's own limits and billing, but it does not attach to an open PR or read its description and discussion, so its findings can still differ from the PR-side round. Use it for the final pre-PR pass on repos where the CLI is the only lane (no PR-side auto-review); keep the default policy for the iterate-and-fix loop. `--deep "<focus>"` (focus text) is early access only. `--light` is gone from `--help` but still accepted as a **legacy alias for the default review** -- it no longer makes anything lighter, so drop it from scripts. `--deep` needs a compatible server; if it is refused, report that the CLI needs updating rather than falling back to a default review silently.

Other review options:

- `--gate <severity>` (v0.9.1) -- exit `2` when the review finds anything at or above `critical`, `major`, `minor`, `trivial` or `info`; the run also reviews the selected diff fresh, without the saved checkpoint. Without it the exit code ignores findings, so this is the flag for a CI or pre-push gate: `coderabbit review --committed --base main --agent --gate major`. Read from `--help`, not exercised here.
- `--use-credits` -- consent to bill this review as usage-based when it exceeds the included allowance (the add-on's `On demand` mode asks for it). **It spends money: never pass it without the user's explicit approval for that run.** Without it, an over-limit run in `On demand` mode charges nothing: in `--agent` mode it ends with an `action_required` record (`status: "awaiting_confirmation"`, the billable file count, the maximum price, and the exact `--use-credits` command) -- relay that to the user. The consent covers only that content; any change needs a fresh approval.
- `--remote owner/repo --base main --source-branch <ref>` (v0.7.7) -- review a GitHub repository server-side without a local checkout; the repo must be installed in the active org, under 300 changed files. `<ref>` is a branch or a full 40-character SHA, never a tag; local selectors (`--dir`, `--committed`, `--uncommitted`, `--include-untracked`, `--base-commit`), `--show-prompts` and local `--config` files do not apply, and nothing is stored for `review findings`.
- `-c, --config <files...>` -- extra instruction files (e.g. `CLAUDE.md`) for one run. They are read from disk even under `--committed`, which takes the reviewed content and the discovered `.coderabbit.yaml` from the Git snapshot -- so commit a config change before testing it with `--committed`.
- `--api-key <key> --region us|eu` -- authenticate one run without storing the key (headless/CI; see the allowlist Caveat). `--region` applies only together with `--api-key`.

## Output Modes

- Default (no mode flag) -- detailed plain-text feedback with fix suggestions and a closing severity summary, non-interactive. In a detected agent environment it prints a notice suggesting `--agent` but stays plain.
- `--agent` -- JSON lines, one event per line (Reading `--agent` Output, below).
- `coderabbit review findings` -- replay cached findings from the most recent local review **that produced findings** (clean sessions are skipped), with no new analysis and **no quota cost** (`--dir <path>` reads a scoped review's cache; up to ten sessions are kept per scope, with no expiry). Use between fix iterations; only re-run a real review to verify at the end. `--clear` (v0.7.7) dismisses the stored findings for the current scope (directory + branch + base) once they are fixed or rejected, so later replays stop surfacing them; it exits `1` if some could not be dismissed (the rest stay dismissed).
- `coderabbit review --show-prompts` -- print the AI prompts from the most recent local review, no new review. It cannot be combined with `--agent`.
- `coderabbit stats` -- statistics from the local review history (`--rebuild` rescans it).
- `coderabbit usage` (`--agent` for one JSON event; `coderabbit --usage` and `coderabbit review --usage` are aliases) -- since v0.8, **the included-review allowance for the current repository**: remaining vs limit over a rolling 1-hour window (`includedReviews.remaining`, `.limit`, `.rollingWindowMs`, and `.fullCapacityInMs`, the time until the bucket is full again). It also reports the billing period (`billingPeriod`): organization, whether usage billing (overage) is active (`usageBillingStatus`), your review count, the usage-based spend so far (`spendCents`), and the period reset date. Each block carries a `state` (`available` when the server answered). v0.9.1 adds the shared review spending cap (`billingPeriod.reviewSpendingCap`: `scope`, `limitCents`, `isActive`) and the credit balances (`reviewCredits.prepaid`, `.legacyCli`), and the event's `status` turns `partial` when any block cannot be read -- observed with both credit lookups `unavailable` (`reason: "lookup_failed"`) and everything else `available`, so judge each block by its own `state`. **Read it before a local loop**: it spends no review. The output does not name the lane, so it does not replace `@coderabbitai rate limit` (below) for a PR-side retry window.
- `coderabbit pullrequest <number|url>` -- **reads CodeRabbit's existing output on a GitHub PR**: no review is run and none is spent. It takes exactly one of `--show-threads` or `--show-prompts`. `--show-threads --agent` returns the inline thread roots as JSON (thread and comment IDs, `path`/`line`, `isResolved`, `isOutdated`, head commit). `--show-prompts` returns one consolidated fix prompt that also carries an **`Outside diff comments:`** section, which a thread-only pass never sees. It needs github.com and a repo installed in the active org; it resolves a bare number from the local `origin`, and a full URL works outside a checkout. Thread handling and resolution stay in the `git-pr` skill.
- `@coderabbitai rate limit` -- **not a CLI command: a PR comment**, and the only on-demand read of the PR-side bucket. Reports the remaining allowance and when the next review becomes available, and [does not consume a review](https://docs.coderabbit.ai/reference/review-commands) (aliases `rate-limit`, `limits`, `quota`). Answers within seconds, even while rate-limited: "Your next review will be available in N minutes" -- the only durable read of that window, since the notice edited into the summary comment can vanish. Post it before opening a loop, and after a bounce instead of probing with triggers. It spends no review, but it is still a PR comment -- a write, subject to the usual approval.

### Reading `--agent` Output

Parse line by line and dispatch on `type`. v0.9.1 also prints a `Tip:` line about `coderabbit skills` on **stderr** before the stream, so read stdout alone, or skip lines that do not start with `{` where the shell merges the two streams:

| `type` | Carries | Use it to |
|--------|---------|-----------|
| `review_context` | `reviewType`, `currentBranch`, `baseBranch`, `workingDirectory`; `expectedDuration` (v0.9.0), a sentence rather than a number | Size the wait: observed "usually 2-6 minutes and can take up to 15", and stopping the command early discards the review -- background it and allow 15+ minutes |
| `status` | `phase`, `status` | Track progress; `review_skipped` means the scope has no changes |
| `heartbeat` | Keep-alive only | Reset idle timers; otherwise ignore |
| `finding` | `severity` (`critical`, `major`, `minor`, `trivial`, `info`, `none`), `fileName`, `codegenInstructions`, `suggestions`, `comment`; `title` and `startLine`/`endLine` (v0.9.0) | Fix per `codegenInstructions` (written for coding agents), falling back to `comment`. Before v0.9.0 `comment` appeared only when `codegenInstructions` was empty |
| `complete` | `status`, `findings`, `outcome` (`completed` on a full run), `message`, `reviewedFiles`, and when relevant `unreviewedFileCount`, `reused`, `persistence` | Decide whether the run counts (rules below); `reviewedFiles` shows what the run actually covered |
| `error` | `errorType`, `recoverable`, `message`, `metadata` | Branch on `errorType`, never on the findings count |

1. **A run is clean only if it exits `0` and its `complete` event reads `status: "review_completed"` with no `outcome: "failed"` and no positive `unreviewedFileCount`.** Since v0.7.7 a failed or incomplete review exits `1` even after emitting findings, and a `review_completed` event can still describe a run that failed or skipped files. `outcome: "completed_with_warnings"` with nothing unreviewed is fine; `status: "review_skipped"` (`findings: 0`, exit `0`) means the scope had no changes. With `--gate` (v0.9.1), exit `2` is a finished review that found something at or above the gate, not a failure.
2. **A bounce is an `error`, not an empty review**: exit `1` with `{"type":"error","errorType":"rate_limit","recoverable":true,"metadata":{"waitTime":"7 minutes"}}` and no findings. Wait out `metadata.waitTime` plus a minute, then re-run once -- a re-run inside the window is answered locally with the same error and the remaining wait (the local wait is capped at 15 minutes), so retrying in a loop buys nothing. `--use-credits` skips that local wait, and bills.
3. **`errorType: "interrupted"`** (v0.9.0, with `elapsedSeconds`) means a signal stopped the run; findings already streamed and the last successful checkpoint are kept -- re-run when ready.
4. **`reused: true` on `complete`** means the server returned an existing review -- not proof that a new review ran.
5. **`persistence.status: "failed"`** (v0.9.0, with an `errorCode`) means the results were not saved locally: keep the findings from this stream, since `review findings` may not replay them, and check local storage permissions and free space. The server-side review can still be complete.
6. **Too many files** fails with an `error` that can carry `candidates` -- narrower `--committed`, `--uncommitted` or `--dir` scopes with estimated file counts -- and a `candidatesNote`. The CLI never picks one or retries: choose one and re-run it.
7. **Security findings** add `category`, CWE, reachability and exploitability. **Unverified findings** are listed and counted separately and carry no fix prompt: inspect them, don't auto-fix them. A review billed as usage-based adds `usageCost`.

## The Local Review-Fix Loop

1. Run `coderabbit review --committed --base {base} --agent` (or `--uncommitted` pre-commit). Background it -- reviews take minutes (v0.9.0's `review_context.expectedDuration` states the range), and killing the command early discards the review.
2. Parse findings; triage by `severity`. Address `critical` and `major` first.
3. **Validate each finding** against the codebase (conventions, actual behavior, project docs). Fix valid ones per `codegenInstructions`; note invalid ones with a one-line rationale for the user.
4. Re-run the same review command to verify fixes -- it reviews the whole scope again, so new findings on untouched code are normal, not a regression. Stop when a clean run (Reading `--agent` Output, rule 1) leaves no valid `critical`/`major` findings, or the hourly bucket is exhausted (`coderabbit usage` shows what remains; a bounce reports the wait -- wait or stop, never hammer).
5. Then push / create the PR -- **once the change is completely finished**, not once the findings are fixed. Where PR-side auto-review and `auto_incremental_review` are on (both default), the push *is* the review request and it reviews whatever the branch holds at that moment: remaining tasks, tests, docs and config belong in the same push, or each leftover costs another PR-side window. The PR-side review (if any) should then come back clean or near-clean.

Commit fixes by what they change, never by what prompted them -- `fix: validate empty page cursor`, not `fix: coderabbit fixes` or `fix: review round 2` (see the `git-commit` skill). Fixes must not add code comments that restate what the code already reads.

Two passes (review, fix, verify) is the normal shape. More than three passes means findings are being treated as noise -- re-evaluate validity or tune config (see `references/configuration.md`).

## Rate Limits (Per Developer, Per Hour)

| Plan | CLI reviews | PR reviews | Chats | Files/review |
|------|-------------|------------|-------|--------------|
| Free | 3 | 1 (summary only) | -- | 150 |
| OSS (public repos) | 3 | 1--10, by repo popularity | 25 | 100--300, by popularity |
| Essentials (was Pro) | 5 | 5 | 50 | 150 |
| Team (was Pro+) | 8 | 8 | 75 | 300 |
| Advanced | 10 | 10 | 100 | 300 |
| Enterprise | 12 | 12 | 100 | 300 |

[Plans](https://docs.coderabbit.ai/management/plans) were renamed in September 2026: Essentials was Pro, Team was Pro+, and Advanced is new. The run-configuration block prints the new name. **Grandfathered Pro+ subscriptions keep 10 reviews/hour** even though they now display as Team, and old Pro/Pro+ subscriptions keep their old throttling ladders. Beyond the hourly allowance, the usage-based add-on bills per reviewed file ($0.25/file, as quoted in the rate-limit notice). **Chats are a separate allowance**: a reply in a CodeRabbit review thread gets a chat answer, which is not a review and does not draw from the PR review bucket (`git-pr` skill, `references/bot-review-loop.md`, Fixes Confirmed in the Thread).

**The table is a CEILING, not the rate you get -- [fair-usage throttling](https://docs.coderabbit.ai/management/rate-limits#fair-usage-limits-policy) sets the effective PR-side allowance below it.** The refill rate comes from the developer's recent PR review count over a **rolling window of either the past 24 hours or the past 7 days**; the plan allowance itself is not changed. Documented ladders by reviews in the last 7 days:

| Plan | Full rate | Steps down to | 1/hr, one at a time |
|------|-----------|---------------|---------------------|
| Essentials | 0--29 -> 5/hr | 30--39 -> 4, 40--49 -> 3, 50--59 -> 2 | 60+ |
| Team | 0--39 -> 8/hr | 40--49 -> 6, 50--59 -> 4, 60--69 -> 2 | 70+ |
| Advanced | 0--49 -> 10/hr | 50--59 -> 8, 60--69 -> 4, 70--79 -> 2 | 80+ |

Enterprise stays at 12/hr unless the contract enrolls in adaptive limits. Observed live on the old Pro plan, crossing a tier mid-loop: 4/hr at round 1, 3/hr two rounds later, then a bounce. On Team, 23 attempts in 7 days still gave the full 8/hr.

Two consequences. **A heavy review week throttles the next one** -- the binding window is days, not the hour, and capacity trickles back as old reviews age out instead of resetting on the hour. And **a bounce is free but pointless**: "a blocked push does not consume a review or delay when your next review becomes available", since capacity is set by earlier *completed* reviews -- yet a refused round is never queued, so extra triggers inside a closed window buy nothing. Probe with `@coderabbitai rate limit` (no review consumed) rather than with triggers -- verified live: after two bounces it still quoted a window closing 54 minutes after the last *completed* review, so the bounces moved nothing. (The run-configuration footer says "review **attempts** over the past 7 days", but the documented ladders count reviews; treat the footer's number, not its noun, as the fact.)

Past the included limit, the [usage-based add-on](https://docs.coderabbit.ai/management/usage-based-addon) decides what happens, and its admin-set mode decides which: `Automatic` keeps reviewing and bills the overage, `On demand` pauses until a seat-holder authorizes each review, `Off` stops until the included allowance resets. Only `Automatic` is a release valve -- and fair-usage spacing is a separate mechanism layered on top.

**Which plan row applies is something you must be told -- but the LIVE allowance is readable in two places:** the "Run configuration" block inside each posted review names the plan and the current reviews-per-hour figure (read it off the most recent review before planning a loop), and `@coderabbitai rate limit` reports the remaining allowance plus next availability on demand, without spending one. The rest is silent about the plan: `coderabbit usage` reports a repository's included allowance and the billing period, but not the tier, and the PR-side API never mentions one. Declare it in the **Code Review Policy** (`git-pr` skill) -- `CodeRabbit plan: team until 2026-10-20, then essentials` -- in the repo's AGENTS.md/CLAUDE.md, or globally for every repo. It decides real behaviour: how many PRs can be non-draft at once, how many rounds to budget, and how much iteration belongs in the CLI lane instead. Re-plan when a trial lapses; a queue tuned for 10 reviews/hour stalls at 5. Undeclared, take the floor from observed behaviour (`bot_bucket`) rather than assuming the best case.

### Where a Bounce Shows Up

**A depleted bucket passes for a clean result in every lane that ignores the lane's own signal.** Each lane does report it -- but never as a failed run, and never in the field most checks look at. Know the tell for the lane you are in:

| Lane | What a bounce looks like | The tell |
|------|--------------------------|----------|
| CLI (`--agent`) | Exit `1` with zero findings -- identical to a failure by exit status, and "no findings" reads as clean | `{"type":"error","errorType":"rate_limit",...,"metadata":{"waitTime":"7 minutes"}}` -- branch on `errorType`, never on the findings count |
| CLI (default output) | The run ends with a limit message and no findings | Read the message; do not treat the empty result as a passing review |
| PR-side | Often **nothing at all** -- no review, no threads, no comment -- plus a green `CodeRabbit` check in the PR's checks list | That check's `description` reads `Review rate limited` (state is `SUCCESS` for every finished outcome): `gh pr checks --json name,state,description`. `Review completed` is the reviewed case, and `Review in progress` (state `PENDING`) means it is still running -- wait rather than trigger |

The PR-side row is the one that misleads most: the checks list shows a tick next to CodeRabbit between passing CI jobs, so the PR reads as reviewed-and-green when the code was never looked at. PR-side handling (windows, re-triggers, the loop) lives in the `git-pr` skill.

**Only the PR itself can tell you when a PR-side round may retry -- ask it with `@coderabbitai rate limit`.** Three dead ends around it, so no one spends the time again:

| Where you might look | What you actually get |
|----------------------|-----------------------|
| `coderabbit usage` | The repository's included allowance (remaining, rolling 1-hour window, time to full) plus the billing period. It does not name the lane and gives no PR-side retry window |
| CI logs behind the check | Nothing: CodeRabbit posts a **commit status**, not an Actions run (`check-runs` is empty, `target_url` is `null`). Log-diagnosable hard limits are a Copilot thing |
| The quota notice on the PR | A per-PR **estimate** that the next summary-comment edit can delete -- two PRs of one developer quoted 48 and 10 minutes 104 seconds apart, while a third PR's review completed 8 minutes later |

What *does* answer it: a `@coderabbitai rate limit` comment (remaining allowance + next availability, no review consumed), and the newest `Review completed` across **all** your open PRs, since the bucket is per developer (`bot_bucket` in `git-pr`'s `references/bot-review-loop.md`). If a sibling PR was reviewed after your bounce, the bucket is open now.

## Configuration

`.coderabbit.yaml` at the repo root governs both PR-side and CLI reviews (the CLI also accepts `-c <file>` for extra instruction files, e.g. `CLAUDE.md`). For typed, well-linted projects the goal is high-level findings only: `profile`, `tone_instructions`, disabled CI-redundant linters, `path_filters` for generated files -- and one automatic PR review per PR (`auto_incremental_review: false`), with later rounds closed in the review threads, whose replies are chats rather than reviews. Validate edits with `coderabbit config validate [file]` (checks against the current official schema, CI-friendly exit codes) before committing. Since v0.7.7 `coderabbit config` also generates and applies a config (`--agent --generate --profile chill` for a proposal) and the CLI discovers `.coderabbit.config.ts` when no YAML exists.

> **Reference**: See `references/configuration.md` for the full schema highlights, config precedence, the low-noise template for typed/linted projects, and PR-side commands (`@coderabbitai review` vs `full review`, pause/resume, config dump).

## Cloud Coding Agent (`coderabbit code`)

**Beta, v0.9.0** (`code handoff` and `code skills import` since v0.8.1; v0.9.1 renames Autopilot to Autofix and adds `goal` and `archive`). `coderabbit code` starts and follows **tasks** that CodeRabbit's Coding Agent runs in the cloud; each instruction runs as a **turn**. Tasks spend agent minutes, billed like web-app tasks, and the first one starts the organization's Coding Agent trial if it has not begun. A person must sign in through the browser and pick an organization with `coderabbit auth org` first: API-key, self-hosted and SSO-workspace logins fail with `unsupported_auth`, and Coding Agent must be enabled for the organization.

| Command | Effect |
|---------|--------|
| `code ls`, `code show <task>`, `code autofix status <task>` | Read only -- rerun freely |
| `code plan <task>` | Shows the latest finalized plan; `--approve` records a reviewer approval (not on a task you started; it does not start implementation) |
| `code new "<prompt>"` | **Starts a billed task** from the branch on `origin` -- unpushed commits and uncommitted changes are not included. `--plan` asks for a plan before any code. v0.9.1: `--goal` starts in Goal mode with the prompt as the goal (`--goal-objective` sets it separately), `--private` hides the task from the rest of the organization, and `--access manual|full` requests repository access (full needs a browser step) |
| `code goal <task>` (v0.9.1) | Reads the goal; `--set <objective>` (1--4,000 characters) sets it and `--clear` removes it. **Editing a completed goal restarts it** |
| `code resume <task>` | Follows a task; with `--agent`, `-m`, `--steer` or `--answer` sends into it and starts or extends billed work |
| `code ask <task> "<question>"` | **Billed** side question; never changes the task |
| `code cancel <task>` | Stops the running turn; **queued messages do not run** -- paused where queue pause has rolled out (resume them from the task page), dropped on automated tasks |
| `code archive <task>` (v0.9.1) | Hides the task from `ls`, makes it read-only and pauses its schedules. Running, starting or stopping turns must finish first; a repeat succeeds, the task can be unarchived, and nothing is canceled or deleted |
| `code push <task>` | **Commits and pushes** the task's changes to its working branch; `--stacked` opens a pull request instead |
| `code autofix on <task>` (or `--pr <n>`) | Autofix (formerly Autopilot; `code autopilot` on v0.9.0, an alias since) publishes the changes when the agent finishes, then **pushes fixes on its own** for CodeRabbit findings, failing required CI and merge conflicts; `off` stops it, `resume` restarts a paused one |
| `code handoff --summary <path>` | **Uploads** a session summary, an optional plan and the discovered Codex or Claude Code transcript into a new task |
| `code skills import [path]` | **Uploads** local skills to the cloud library (`--scope organization` shares them with the org) |

Driving it from an agent:

1. **Pass `--agent` right after the command name** (`coderabbit code new --agent "<prompt>"`, before `--plan`). Under a detected agent environment with piped stdout the CLI emits JSON anyway (observed under Claude Code), but only an explicit `--agent` turns argument errors into JSON.
2. **The last record decides** -- branch on its `type`, then its `status` or `action`; the exit code is a cross-check (`0` complete, `1` error or a login/authorization action, `3` a question to answer, `4` rerun `resume` -- or, after `ask`, read the answer in the web app). The first record is always a `beta_notice` status.
3. **Never rerun a command after it wrote a commit-point record** (`task_submitted`, `creating_task`, `message_sent`, `steer_sent`, `answer_sent`, `question_sent`): the rerun creates a second billed task or question. Follow the task with `coderabbit code resume --agent <task>` instead, and pass explicit task IDs, never `--last`, which can change between runs.
4. **Get the user's OK before** `push`, `plan --approve`, `autofix on`/`resume`, `goal --set`/`--clear`, `archive`, `cancel`, or pushing what a failed or canceled turn left behind. Hand `authorize_github` URLs, login commands and sensitive questions to the user; never open or answer them yourself. Where PR-side auto-review is on, each Autofix or `push` commit to a PR branch is also a PR review request (`git-pr` skill).

> **Reference**: See `references/cloud-tasks.md` for the record stream, per-command outcomes, safe reruns and wait limits

## Key Gotchas

1. **`--committed` needs commits, `--uncommitted` needs a dirty tree** -- reviewing the wrong scope silently reviews nothing (`review_skipped`); match the scope to the checkpoint.
2. **`coderabbit review findings` replays, it does not re-review** -- after fixing, cached findings still show; only a fresh review verifies fixes. It also skips clean sessions: even after a clean verify run, it replays the older findings, which is not a regression.
3. **The quota is per-hour, not per-day** -- a "limit reached" message means wait for the window, not stop for the day. Plan verify runs so the final pass fits the bucket. In `--agent` mode the bounce is a `rate_limit` error line at exit `1` -- indistinguishable from a real failure by exit status alone, and a findings-count check reads it as clean. Branch on `errorType` and take the window from `metadata.waitTime`.
4. **CLI reviews and PR reviews draw from separate buckets** -- burning CLI reviews locally does not reduce the PR-side allowance, which is the point of the local-first flow.
5. **Org context needs matching access** -- on a repo not linked to your CodeRabbit org, reviews run in limited free mode (no learnings/org context); results differ from PR-side reviews on the same code.
6. **`--agent` emits JSON lines, not a JSON document** -- parse line-by-line; do not `JSON.parse` the whole output.
7. **`--plain` depends on the binary; `--type` and `--light` are only hidden** -- v0.7 through v0.9.0 reject `--plain` (`error: unknown option`, exit `1` since v0.8), and v0.9.1 accepts it again as a hidden no-op, so a script that carries it passes on one install and fails on the next. `--type <all|committed|uncommitted>` still runs a review although `--help` omits it, and fails next to `--committed`/`--uncommitted`; `--light` still parses as an alias for the default review since v0.8, and v0.9.1 takes the removed `--prompt-only` as `--agent`. Use `--committed`/`--uncommitted`, rely on the plain default, and treat `--deep` as the real depth switch.
8. **A PR-side rate limit is silent and green** -- no review, no threads, usually no comment, and a passing `CodeRabbit` check whose `description` reads `Review rate limited`. Nothing about the PR looks wrong, so an unreviewed PR gets reported as reviewed. Read that description before concluding anything from a quiet PR-side round (Where a Bounce Shows Up).
9. **A quoted retry window is an estimate, and it is not durable** -- it is edited into the summary comment and a later edit can remove it, while different PRs quote wildly different numbers at the same moment. Capture it when you see it, take the smallest one visible across your PRs, and treat a sibling PR's `Review completed` as the real all-clear (Where a Bounce Shows Up). Don't wait out the largest number you can find.
10. **The push is the PR-side review request** -- with auto-review plus `auto_incremental_review`, every push spends a PR-side review of whatever is on the branch, from a bucket that is **per developer, not per PR**. Finish the whole change locally, then push once; the local lane (separate bucket) is where iteration belongs. With incremental reviews off (the template in `references/configuration.md`), fix pushes spend nothing and the thread replies verify the fixes instead.
11. **A rate-limited attempt can leave its commits looking reviewed** -- once the window reopens, a plain `@coderabbitai review` over unchanged commits answers "does not re-review already reviewed commits" and the round silently never happens, so the recovery trigger after a bounce is **`@coderabbitai full review`**. It does not always stick, though: observed on this repo, the *next push* after a bounce resumed its incremental range from the last **completed** review and did re-cover the two skipped commits unprompted. Read the review's own "Commits" block to see which range it actually took, rather than assuming either way. The same forcing form is the fix for a wedged **"Review queued"** (observed stuck 2+ hours): nudge after ~1 hour instead of waiting it out.
12. **A clean local pass is one sample, not proof** -- runs vary: observed here, a second run on an unchanged scope reported six findings in code the first run had passed clean. Budget for that, and let the PR-side round have the final say where it exists. `--fresh` (v0.8.2) reviews without the saved checkpoint context.
13. **`coderabbit code` is billed cloud work, not a review** (v0.9.0, beta) -- `new`, `ask` and sends through `resume` spend agent minutes, `push` and Autofix (formerly Autopilot) push to the repository, and `handoff` and `skills import` upload local material. A rerun after a commit-point record duplicates the billed work. The top-level `handoff` from v0.8.0 survives only as a hidden alias of `code handoff`. Approve each state-changing run (`references/allowlist.md`).
14. **A Homebrew install has no `cr` and no self-update** -- the cask links only `coderabbit`, and `coderabbit update` defers to `brew upgrade coderabbit` (v0.9.0). A `cr ...` line copied from docs fails there with `command not found`; write the full name.
15. **Exporting `CODERABBIT_API_KEY` authenticates nothing** -- the CLI reads no API-key environment variable (v0.9.0 `--help`), so a CI step that only exports it runs under whatever login is stored, or none. Pass `--api-key "$CODERABBIT_API_KEY"`, or store the key with `coderabbit auth login --api-key` in an earlier step of the same environment.
16. **`status: "review_completed"` is not proof the review completed** -- a failed run or one that skipped files can still end with it. Check the exit code, `outcome` and `unreviewedFileCount` before calling a run clean (Reading `--agent` Output).
17. **Autofix means two different things** -- in the CLI, `code autofix` (v0.9.1) is the loop formerly called Autopilot, which keeps pushing fixes until the PR is ready; on a PR, `@coderabbitai autofix` is a legacy alias of the one-off **Fix comments** repair, and Autofix itself is still switched on with `@coderabbitai autopilot` (`references/configuration.md`). Name the exact command, not the feature, when asking the user to approve one.

> **Reference**: See `references/configuration.md` for `.coderabbit.yaml` tuning and PR commands
> **Reference**: See `references/cloud-tasks.md` for driving `coderabbit code` cloud tasks from an agent
> **Reference**: See `references/allowlist.md` for auto-approval patterns
> **Reference**: PR-side review loops, thread handling: `git-pr` skill (`references/bot-review-loop.md`)
