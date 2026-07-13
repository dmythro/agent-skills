# CodeRabbit Configuration

Tuning CodeRabbit for low-noise, high-signal reviews via `.coderabbit.yaml`, plus the PR-side commands that control review behavior. Optimized here for strongly-typed, well-linted projects where style/lint feedback is redundant and only high-level findings (logic, security, contracts) earn their place.

## Config Precedence

From most to least authoritative:

1. **Global overrides** (org admins, CodeRabbit dashboard) -- enforced everywhere
2. **Repository `.coderabbit.yaml`** -- the copy on the **feature branch under review**; when present it replaces repo UI settings entirely (no per-key merge)
3. **Central configuration** -- `.coderabbit.yaml` in the organization's `coderabbit` repository
4. **Repository UI settings** (app.coderabbit.ai)
5. **Organization settings**
6. Built-in defaults

By default the highest-priority source wins whole (no merging); with `inheritance: true` parent-layer settings merge in -- global overrides still win either way.

Dump the fully resolved config (with per-setting source annotations) by commenting on any PR:

```text
@coderabbitai configuration
```

The CLI reads the same `.coderabbit.yaml`; `coderabbit review -c <file>` adds extra instruction files (e.g. `CLAUDE.md`) for one run.

## Low-Noise Template (Typed + Linted Projects)

Copy-paste starting point. Assumes CI already runs the type checker and linters, so CodeRabbit should skip everything they cover:

```yaml
# yaml-language-server: $schema=https://coderabbit.ai/integrations/schema.v2.json
language: "en-US"
tone_instructions: >-
  Be brief and direct. Comment only on bugs, logic errors, security issues,
  data loss, and breaking API changes. Skip style, naming, formatting,
  imports, and documentation nits - linters and the type checker cover those.

reviews:
  profile: chill                  # quiet | chill | assertive; quiet = only the most important feedback
  high_level_summary: true
  poem: false
  review_status: false
  collapse_walkthrough: true
  sequence_diagrams: false
  changed_files_summary: false
  suggested_labels: false
  suggested_reviewers: false
  assess_linked_issues: false
  related_issues: false
  related_prs: false
  estimate_code_review_effort: false

  auto_review:
    enabled: true
    drafts: false                 # default; reviews start when marked ready
    auto_incremental_review: true # re-review each push; false = once per PR

  path_filters:
    - "!**/dist/**"
    - "!**/*.min.js"
    - "!**/*.gen.ts"
    - "!**/*.snap"
    - "!bun.lock"
    - "!package-lock.json"
    - "!pnpm-lock.yaml"

  path_instructions:
    - path: "**/*.{ts,tsx}"
      instructions: >-
        This codebase uses strict TypeScript and a full lint suite in CI.
        Do not comment on type style, formatting, or lint-level issues.
        Focus on: logic correctness, edge cases, race conditions, error
        handling, security (injection, authz, secrets), and API contract
        or breaking behavior changes.

  tools:                          # CI already runs these; avoid duplicate findings
    eslint: {enabled: false}
    biome: {enabled: false}
    oxlint: {enabled: false}
    markdownlint: {enabled: false}
    yamllint: {enabled: false}
    languagetool: {enabled: false}
    # keep semgrep, gitleaks, actionlint, shellcheck enabled (default true)

  finishing_touches:
    docstrings: {enabled: false}
    unit_tests: {enabled: false}

knowledge_base:
  learnings:
    scope: auto
```

Escalate `profile: chill` to `quiet` if reviews still feel noisy; `quiet` restricts output to only the most important feedback. `assertive` is explicitly nitpicky -- never use it for this goal.

## Noise Levers, Ranked by Impact

1. **`reviews.profile`** -- `quiet` < `chill` (default) < `assertive` in feedback volume.
2. **Disable CI-redundant tools** -- all ~50 `reviews.tools.*` default to `enabled: true`; CodeRabbit runs them on the diff and posts their output. If CI runs eslint/biome/tsc, CodeRabbit re-running them only duplicates CI. `languagetool` (grammar in comments/docs) is a common noise source. Keep security tools (`semgrep`, `gitleaks`) -- CI often lacks them.
3. **`tone_instructions`** -- max 250 chars, applies to reviews and chat.
4. **`path_instructions`** -- per-glob focus instructions (up to 20,000 chars each); `path: "**"` works for repo-wide instructions.
5. **`path_filters`** -- `!`-prefixed globs remove generated/vendored files from review entirely.
6. **Presentation toggles** -- `poem`, `sequence_diagrams`, `changed_files_summary`, `review_status`, `collapse_walkthrough` shrink each review's comment bulk without affecting findings.
7. **Learnings** -- reply to an unwanted comment with `@coderabbitai` + why it should not be flagged; the learning persists to future reviews (manage at app.coderabbit.ai/learnings). `knowledge_base.code_guidelines` auto-ingests CLAUDE.md/.cursorrules-style files -- encode "no style nits" there too.

## Auto-Review Behavior

Key `reviews.auto_review` knobs:

| Key | Default | Effect |
|-----|---------|--------|
| `enabled` | `true` | Auto-review new PRs |
| `drafts` | `false` | Draft PRs are NOT reviewed until marked ready |
| `auto_incremental_review` | `true` | Re-review each push (incremental); `false` = one review per PR |
| `auto_pause_after_reviewed_commits` | `5` | **Silently pauses** auto-reviews after 5 reviewed commits; `0` disables the pause |
| `ignore_title_keywords` | `[]` | Skip PRs by title keyword (e.g. `["wip"]`) |
| `labels` | `[]` | Label gate; `"!no-review"` = skip labeled PRs |
| `base_branches` | `[]` | Regex patterns for extra base branches to review |

The `auto_pause_after_reviewed_commits` default is a common "CodeRabbit stopped reviewing" cause on long-running PRs -- it is a quota guard, not a failure. Resume with `@coderabbitai resume` or raise/zero the setting.

## PR-Side Commands

Posted as PR comments (except `ignore`):

| Command | Effect |
|---------|--------|
| `@coderabbitai review` | **Incremental** review -- new changes since the last review only |
| `@coderabbitai full review` | Full from-scratch review of all files in the PR |
| `@coderabbitai pause` / `resume` | Stop / restart automatic reviews on this PR |
| `@coderabbitai ignore` | In the PR **description**: permanently disable auto-review for the PR |
| `@coderabbitai resolve` | Resolve all CodeRabbit review threads |
| `@coderabbitai autofix` | Apply fixes for unresolved findings (commit or stacked PR) |
| `@coderabbitai configuration` | Dump the resolved config with per-setting sources |
| `@coderabbitai generate configuration` | Open a PR adding a `.coderabbit.yaml` |

Every review run -- automatic incremental on push, `@coderabbitai review`, or `full review` -- consumes one PR review from the hourly allowance. Prefer incremental; reserve `full review` for after large refactors/rebases or when earlier reviews predate significant context.

## Knowledge Base

```yaml
knowledge_base:
  opt_out: false                # true disables the knowledge base entirely
  code_guidelines:
    enabled: true               # auto-ingests CLAUDE.md, .cursorrules, etc.
    filePatterns: []            # add custom guideline files, e.g. "docs/STANDARDS.md"
  learnings:
    scope: "auto"               # local | global | auto (repo-local for public, org-wide for private)
    approval_delay: 0           # 0-30 days; >0 gates new learnings behind admin approval
```

Bulk-import standards as learnings: `@coderabbitai add a learning using docs/coding-standards.md`.
