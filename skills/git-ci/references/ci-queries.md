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

A passing check can still mean "did nothing" -- `description` is the only field that says so (CodeRabbit reports `Review rate limited` as `SUCCESS`).

```bash
# From the checks list
gh pr checks {number} --json name,state,description --jq '
  .[] | select(.name|ascii_downcase|startswith("coderabbit"))'

# From the commit status API, pinned to the PR head (check-runs does NOT contain it)
gh api repos/{owner}/{repo}/commits/"$(gh pr view {number} --json headRefOid --jq .headRefOid)"/status --jq '
  [.statuses[] | select(.context|ascii_downcase|startswith("coderabbit"))] | max_by(.updated_at) | {state, description, updated_at}'
```

The `gh api` form is **not** covered by this skill's allowlist, which keeps every `gh api` call manual (`references/allowlist.md`); the `git-pr` skill allowlists it as a GET-only pattern (`repos/*/commits/*/status`). The `gh pr checks` form above needs no exception.

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
