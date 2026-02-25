# CI/CD Query Patterns

Advanced CI/CD query patterns for GitHub Actions and GitLab CI.

## GitHub: Failing Checks

```bash
# Names of failing checks
gh pr checks {number} --json name,conclusion --jq '
  .[] | select(.conclusion == "FAILURE") | .name'

# All checks with status
gh pr checks {number} --json name,state,conclusion,startedAt,completedAt
```

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

# JSON output
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
