# CI/CD Query Patterns

Advanced CI/CD query patterns for GitHub Actions and GitLab CI.

## GitHub: Failing Checks

`gh pr checks` has **no `conclusion` field** (unknown fields abort the command); its verdict fields are `state` and the coarser `bucket`.

```bash
# Names of failing checks
gh pr checks {number} --json name,bucket --jq '
  .[] | select(.bucket == "fail") | .name'

# All checks with status and each app's own summary line
gh pr checks {number} --json name,state,bucket,description,startedAt,completedAt
```

## GitHub: Review-Bot Check Verdict

A passing check can still mean "did nothing" -- only the verdict says so. CodeRabbit reports `Review rate limited` as a `neutral` check run by default, or as a `SUCCESS` commit status where the legacy surface is kept.

```bash
# Check run (default surface), pinned to the PR head: the verdict is the first summary line
gh api repos/{owner}/{repo}/commits/"$(gh pr view {number} --json headRefOid --jq .headRefOid)"/check-runs --paginate --slurp | jq '
  [.[].check_runs[] | select(.app.slug=="coderabbitai")] | max_by(.completed_at // .started_at)
  | if . == null then empty else {status, conclusion, verdict: ((.output.summary // "") | split("\n")[0])} end'

# Legacy commit status, pinned to the PR head
gh api repos/{owner}/{repo}/commits/"$(gh pr view {number} --json headRefOid --jq .headRefOid)"/status --paginate --slurp | jq '
  [.[].statuses[] | select(.context|ascii_downcase|startswith("coderabbit"))] | max_by(.updated_at)
  | if . == null then empty else {state, description, updated_at} end'

# From the checks list: the status description, never a check run's verdict
gh pr checks {number} --json name,state,bucket,description --jq '
  .[] | select(.name|ascii_downcase|startswith("coderabbit"))'
```

The `gh api` forms are **not** covered by this skill's allowlist, which keeps every `gh api` call manual (`references/allowlist.md`); the `git-pr` skill allowlists both as GET-only patterns (`repos/*/commits/*/status`, `repos/*/commits/*/check-runs`). The `gh pr checks` form needs no exception.

## GitHub: Required Checks Status

```bash
gh pr view {number} --json statusCheckRollup --jq '
  .statusCheckRollup[] | {name: .name, status: .status, conclusion: .conclusion}'
```

## GitHub: Workflow Listing

```bash
# List all workflows
gh workflow list --json id,name,state

# Runs for a specific workflow
gh run list --workflow ci.yml --limit 5 --json status,conclusion,headBranch

# Workflow ID lookup
gh workflow list --json name,id --jq '.[] | "\(.id) \(.name)"'
```

## GitHub: Run Details

```bash
# Full run details with jobs
gh run view {run_id} --json jobs,status,conclusion,displayTitle

# Failed job logs only
gh run view {run_id} --log-failed

# All run logs (verbose)
gh run view {run_id} --log
```

## GitHub: Cache Management

```bash
# List caches
gh cache list

# Cache usage
gh cache list --json key,sizeInBytes --jq '
  [.[] | .sizeInBytes] | add | . / 1048576 | round | "\(.)MB total"'
```

## GitHub: Rulesets

```bash
# List rulesets
gh ruleset list

# View specific ruleset
gh ruleset view {id}

# Check branch compliance
gh ruleset check {branch}
```

## GitHub: PRs with Failing CI

```bash
gh pr list --json number,title,statusCheckRollup --jq '
  .[] | select(.statusCheckRollup | any(.conclusion == "FAILURE"))
  | "\(.number) \(.title)"'
```

---

## GitLab: Pipeline Status

```bash
# Current branch pipeline
glab ci status

# Full pipeline detail
glab ci get

# Live watch
glab ci status --live
```

## GitLab: Pipeline Listing

```bash
# Recent pipelines
glab ci list

# View specific pipeline
glab ci view {pipeline_id}
```

## GitLab: Job Logs

```bash
# Trace a specific job
glab ci trace {job_id}
```

## GitLab: Pipeline Jobs

```bash
# List jobs in a pipeline
glab api projects/{project_id}/pipelines/{pipeline_id}/jobs | jq '[.[] | {id:.id,name:.name,status:.status,stage:.stage}]'

# Failed jobs only
glab api projects/{project_id}/pipelines/{pipeline_id}/jobs | jq '[.[] | select(.status=="failed") | {id:.id,name:.name,stage:.stage}]'
```

## GitLab: Merge Readiness

```bash
glab mr view -F json | jq '{merge_status:.merge_status,conflicts:.has_conflicts,blocking:.blocking_discussions_resolved,draft:.draft}'
```

## GitLab: Variables

```bash
glab variable list
```
