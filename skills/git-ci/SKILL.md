---
name: git-ci
description: >-
  CI/CD status queries for GitHub Actions (gh) and GitLab CI (glab). Check
  pipeline status, failing jobs, workflow runs, job logs, and merge readiness.
  Use when checking CI status, debugging build failures, viewing job logs, watching
  pipeline progress, or configuring gh/glab CI read-only allowlists.
  Not for PR workflows (git-pr), git commits (git-commit), or deploy/release ops
---

# CI/CD Status Queries

**Query CI/CD pipelines and check merge readiness across GitHub Actions and GitLab CI.** All recipes use minimal field sets for token efficiency. Covers pipeline status, failing jobs, run logs, workflow management, and merge readiness assessment.

## When to Use

- **Checking CI status** -- "are checks passing?", "what's failing?", pipeline status
- **Watching CI runs** -- wait for completion, fail-fast on errors
- **Debugging failures** -- view logs for failing jobs, identify flaky tests
- **Assessing merge readiness** -- all checks green, reviews approved, no conflicts
- **Listing workflow runs** -- recent pipelines, specific workflow history
- **Configuring CI allowlists** -- auto-approval patterns for read-only CI commands

## Critical Rules

1. **Use `gh pr checks` (not `gh run`) for current-branch CI status.** The `pr checks` subcommand maps directly to the PR's required status checks.
2. **Use `glab ci status` for current-branch CI in GitLab.** It shows the pipeline for the current branch without needing a pipeline ID.
3. **Always use `--json` with `gh` to filter output fields.** Full JSON output wastes tokens. `gh pr checks` accepts exactly `bucket`, `completedAt`, `description`, `event`, `link`, `name`, `startedAt`, `state`, `workflow` -- **there is no `conclusion` field here** (that one belongs to `gh run` and to `statusCheckRollup`), and an unknown field aborts the command with `Unknown JSON field`.
4. **A green check is not always a completed job -- read `description`.** App and bot checks report `state: SUCCESS` / `bucket: pass` for outcomes that did no work: CodeRabbit posts `Review rate limited` as a **successful** check when it never reviewed the code. `description` is where the outcome lives, it is not returned by default, and `gh pr view --json statusCheckRollup` drops it entirely. See Review-Bot Checks below before calling a PR reviewed or merge-ready.
5. **Use commands exactly as shown in this skill.** The commands below are designed to match auto-approval allowlist patterns. Improvising flags may trigger permission prompts.

---

## Provider Detection

```bash
git remote get-url origin
```

| Remote URL contains                | Provider | CLI    | CI system        |
|------------------------------------|----------|--------|------------------|
| `github.com`                       | GitHub   | `gh`   | GitHub Actions   |
| `gitlab.com` or self-hosted GitLab | GitLab   | `glab` | GitLab CI        |

If ambiguous or both present, ask the user.

**The `glab` recipes follow GitLab's documentation and are not exercised against a live instance** -- the GitHub ones are. Confirm flags with `glab <command> --help` before depending on one unattended.

---

## CI Status (Current Branch)

**GitHub:**
```bash
gh pr checks --json name,state,bucket,description
```

`state` is GitHub's raw verdict (check runs and legacy status contexts use different vocabularies -- `SUCCESS`, `FAILURE`, `PENDING`, `SKIPPED`, ...); `bucket` collapses it to exactly `pass`, `fail`, `pending`, `skipping`, or `cancel`. `description` carries the app's own summary line -- always request it: it is the only field that separates a check that did its job from one that merely reported (Critical Rule 4). Branch on the JSON, not on the exit status: `gh pr checks` exits `8` while checks are pending and non-zero when any fails or none are reported.

**GitLab:**
```bash
glab ci status          # current branch, one line per job
glab ci get             # full detail for the branch's pipeline
```

## Review-Bot Checks (GitHub)

**Review bots publish their outcome as a commit status, always green.** In the checks list it sits among the CI jobs as `CodeRabbit -- Review rate limited` with a tick, which reads as "everything passed" while meaning the code was never reviewed.

```bash
# Read the bot rows with their description (name + state + the line that actually matters)
gh pr checks --json name,state,bucket,description --jq '.[] | select(.name|ascii_downcase|startswith("coderabbit"))'
```

| `state` / `bucket`     | `description`         | Reality                                             |
|------------------------|-----------------------|------------------------------------------------------|
| `PENDING` / `pending`  | `Review in progress`  | Running right now -- the PR is not reviewed *yet*    |
| `SUCCESS` / `pass`     | `Review completed`    | The bot reviewed this commit                         |
| `SUCCESS` / `pass`     | `Review rate limited` | The hourly bucket was empty -- **nothing was reviewed**, and nothing is queued |
| `SUCCESS` / `pass`     | `Review skipped: <reason>` | Excluded by config (`draft pull request`, `reviews are disabled for this base branch`) -- nothing was reviewed |

`gh pr checks` uppercases `state` and adds the lowercase `bucket`; the underlying REST commit status returns lowercase (`success`) and no bucket at all. Across 25 PRs of one account, 15 of 62 rounds came back `Review rate limited` -- a green check next to the CI jobs meaning nothing was reviewed, roughly a quarter of the time.

`state` tells running from finished, nothing more: both finished outcomes are `success`.

Two surfaces make this invisible, so avoid both for bot checks: `gh pr view --json statusCheckRollup` returns `context`/`state`/`targetUrl` only (no description), and `gh api repos/{owner}/{repo}/commits/{sha}/check-runs` does not list it at all -- it is a commit *status*, readable per sha via `gh api repos/{owner}/{repo}/commits/{sha}/status`. Handling the rate-limited case (windows, re-triggers, the review loop) is the `git-pr` skill's `references/bot-review-loop.md`.

## Watch Until Complete

**GitHub:**
```bash
gh pr checks --watch --fail-fast
```

**GitLab:**
```bash
glab ci status --live
```

## Recent Pipeline/Workflow Runs

**GitHub:**
```bash
gh run list --json databaseId,displayTitle,status,conclusion,headBranch,event --limit 10
```

**GitLab:**
```bash
glab ci list
```

## Specific Run Details

**GitHub:**
```bash
gh run view {run_id} --json jobs,status,conclusion,displayTitle
```

**GitLab:**
```bash
glab ci view {pipeline_id}
```

## Run Logs

**GitHub:**
```bash
gh run view {run_id} --log-failed
```

**GitLab:**
```bash
glab ci trace {job_id}
```

---

## Merge Readiness (Current Branch)

**GitHub:**
```bash
gh pr view --json mergeable,reviewDecision,statusCheckRollup,isDraft,mergeStateStatus
```

Fields:
- `mergeable` -- `MERGEABLE`, `CONFLICTING`, or `UNKNOWN`
- `reviewDecision` -- `APPROVED`, `CHANGES_REQUESTED`, `REVIEW_REQUIRED`, or empty
- `isDraft` -- boolean
- `mergeStateStatus` -- `CLEAN`, `BLOCKED`, `BEHIND`, `DIRTY`, `UNSTABLE`

`statusCheckRollup` answers "did the checks pass", never "did each check do its job" -- it carries no `description`, so a bot check that reported a rate limit looks identical to one that reported a completed review. Pair it with the `gh pr checks --json ...,description` call above whenever a review bot gates the merge.

**GitLab:**
```bash
glab mr view -F json | jq '{merge_status:.merge_status,conflicts:.has_conflicts,blocking:.blocking_discussions_resolved,draft:.draft}'
```

---

## Variables and Secrets

**GitHub:**
```bash
gh variable list
gh secret list
```

**GitLab:**
```bash
glab variable list
```

---

## CI Queries Reference

> **Reference**: See `references/ci-queries.md` for advanced patterns: failing check extraction, required checks, workflow listing, cache management, rulesets.

---

## Allowlist

> **Reference**: See `references/allowlist.md` for tiered `Bash(command:*)` patterns covering all read-only CI operations -- safe to auto-approve in Claude Code `settings.json` or OpenCode config.
