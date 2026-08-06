# Copilot Code Review Configuration

Managing GitHub Copilot code review per repo: billing awareness, detecting and configuring automatic reviews, tuning review focus, and the Copilot CLI as a local pre-push option. Copilot reviews are **billable, metered actions** (since June 2026 they consume AI credits or legacy premium requests, plus GitHub Actions minutes on private repos) -- configuration directly controls cost.

## Billing Models (Detect Before Budgeting Rounds)

Two coexisting models -- which one applies changes the cost calculus:

| Model | Who | Cost per review |
|-------|-----|-----------------|
| **Legacy premium requests** | Annual Pro/Pro+ plans from before 2026-06-01, until expiration | **13 premium requests** flat (Pro: 300/month = up to ~23 reviews; the pool is shared with all premium features); overage $0.04/request = $0.52/review if a premium-request budget is active |
| **AI credits** (current) | Everyone else | Token-metered (no fixed price; Medium effort costs more) + **Actions minutes** on private repos |

Notes:

- Counters reset the **1st of each month 00:00 UTC**, not on the subscription anniversary.
- **Every re-request bills the same as a first review** -- no incremental discount. This is why review rounds are budgeted (see `bot-review-loop.md`).
- Copilot Free includes no *personal* code review (org-enabled review can still cover unlicensed members, billed to the org). Legacy annual plans drop to Free at expiration.
- Legacy overage pitfall: the budgets UI now creates "Copilot AI credits" budgets; those may not gate legacy premium-request overage. A budget on the premium-request SKU (or "All Premium Request SKUs") is the one that unblocks legacy overage, and entitlement sync can lag 24-48h after budget changes.
- **Hard model rate limits are a separate axis from spend** ([docs](https://docs.github.com/copilot/concepts/rate-limits)): heavy use can hit a weekly per-model cap that fails reviews without consulting the budget or remaining allowance at all. The PR shows only the generic "Copilot encountered an error" comment; the review run's Actions log (workflow `Copilot`) carries the real error -- `SessionModelError ... You've reached your weekly rate limit. Please wait for your limit to reset on <date>` (errorType `rate_limit`, HTTP 429). Every re-request before that reset fails identically and still burns Actions minutes; diagnosis recipe: `bot_fail_diag` in `bot-review-loop.md`.

**Check usage via API** (requires the `user` scope: `gh auth refresh -h github.com -s user`). Note `gh api` only auto-fills `{owner}`/`{repo}`/`{branch}` -- the username must be substituted explicitly:

```bash
# Legacy: premium request usage (rows here = legacy billing)
gh api "/users/$(gh api user --jq .login)/settings/billing/premium_request/usage?year=$(date +%Y)&month=$(date +%-m)"
# Current: AI credit usage (rows here = AI credits billing)
gh api "/users/$(gh api user --jq .login)/settings/billing/ai_credit/usage?year=$(date +%Y)&month=$(date +%-m)"
```

Responses report consumption (`discountQuantity` = used from included allowance, `netQuantity` = billed overage), not remaining -- compute remaining against the plan allowance. Whichever endpoint returns rows tells you which billing model the account is on -- but only for **personally billed** usage: org-paid seats bill the org, so empty responses on both endpoints mean org-managed billing (query the org endpoints), not proof of either model.

## Automatic Review: Detect, Then Configure

Auto-review can come from **two independent sources**:

1. **Repo/org ruleset** -- the `copilot_code_review` branch rule (API-visible)
2. **Personal setting** -- user Copilot settings > "Automatic Copilot code review" (Pro/Pro+/Max; **no API**, invisible to detection)

### Read the Ruleset (Read-Only Detection)

```bash
# All rules active on the default branch (merges repo + org rulesets).
# Keep the REST path unquoted so it matches the allowlist patterns:
gh api repos/{owner}/{repo}/rules/branches/$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name) --jq '[.[] | select(.type=="copilot_code_review")]'
```

Result semantics:

- **Rule present** -- auto-review is on; `parameters.review_on_push` (re-review each push) and `parameters.review_draft_pull_requests` tell you which events fire it.
- **Empty array** -- no *ruleset* auto-review, but the personal setting may still fire one. Fall back to observation: a pending `Copilot` entry in REST `requested_reviewers` or an unrequested bot review on a fresh PR (see `bot-review-loop.md`).
- **HTTP 403 "Upgrade to GitHub Pro..."** -- rulesets are unavailable on Free-plan private repos; ruleset auto-review cannot exist here, but the personal setting still can. Use observation.

### Write the Ruleset (Manual Approval)

Enable auto-review on the default branch, one review per PR (no re-review on push, no drafts) -- the cost-efficient shape:

```bash
gh api -X POST repos/{owner}/{repo}/rulesets --input - <<'EOF'
{
  "name": "copilot-auto-review",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "copilot_code_review",
      "parameters": { "review_on_push": false, "review_draft_pull_requests": false } }
  ]
}
EOF
```

Update with `PUT /repos/{owner}/{repo}/rulesets/{ruleset_id}`; org-wide via `/orgs/{org}/rulesets` with `conditions.repository_name` patterns. Both parameters default to `false`.

Cost guidance: keep `review_on_push: false` (each push-triggered review bills fully; re-request deliberately instead) and `review_draft_pull_requests: false` (draft = free workspace; review fires on ready-for-review). A ruleset rule **overrides a personal disable** -- to fully stop auto-reviews, clear both.

### Effort Level (UI-Only)

Repo Settings > Copilot > Code review > "Review effort level". **Low** (default): fast, cost-efficient, targeted at bugs/security/consistency. **Medium** (preview): routes to a higher-reasoning model, for complex logic / security-sensitive / cross-service changes -- consumes more credits and Actions minutes (no published ratio). No API; org admins can set a default for unconfigured repos. Keep Low unless the repo genuinely needs deep reviews and the budget allows.

## Focusing Reviews (Noise and Cost Reduction)

Copilot code review honors, merged together:

- **`AGENTS.md`** (repo root only) -- shared with coding agents; a good home for the project's Code Review Policy section
- **`.github/copilot-instructions.md`** -- repo-wide instructions (no size limit since June 2026, but keep any file under ~1,000 lines or quality degrades)
- **`.github/instructions/*.instructions.md`** -- path-scoped via `applyTo:` glob frontmatter; `excludeAgent: "code-review"` hides a file from review (or `"coding-agent"` to make it review-only)
- **Org-level custom instructions** -- included automatically

For typed/linted projects, direct the reviewer away from what CI already covers:

```markdown
<!-- .github/copilot-instructions.md -->
## Code Review
- Do not comment on formatting, naming, import order, or style; CI linters cover those.
- Do not suggest type annotations; strict TypeScript is enforced by CI.
- Focus on: logic errors, edge cases, race conditions, error handling,
  security (injection, authz, secrets), breaking API/contract changes.
- Keep comments brief: one sentence of problem, one suggested fix.
```

**Instructions are best-effort, not guaranteed** -- Copilot may still occasionally post excluded categories; that is documented behavior, not a config error. Ignored instruction types: comment formatting/UX, PR-overview changes, merge-blocking, external URLs ("follow the standards at https://..."), and vague directives. Use 10-20 short imperative directives.

### Content Exclusions

Repo/org Copilot **content exclusion** settings apply to code review (since June 2026): excluded paths are not reviewed -- a real cost lever for generated code. Already excluded by default (no config needed): lockfiles (`package-lock.json`, `bun.lock`, `**/*.lock`, `go.sum`, ...), `**/*.min.js`, `**/*.map`, `**/*.d.ts`, `**/node_modules/**`, `**/dist/**`, `**/generated/**`, `**/vendor/**`, `**/coverage/**`, `**/*.svg`, `**/*.log`.

## Copilot CLI: Local Pre-Push Reviews (Secondary Option)

For Copilot-preferring repos without CodeRabbit, the Copilot CLI (GA since Feb 2026) reviews local changes before push -- on legacy billing a single-prompt CLI review costs ~1 premium request vs 13 for a PR-side review:

```bash
npm install -g @github/copilot     # or: brew install copilot-cli
copilot                            # interactive: /review reviews staged+unstaged changes
# Headless, committed-changes review vs base:
copilot -p "Review the diff vs origin/main: logic errors, edge cases, security. Skip style; CI lints." -s --allow-tool 'shell(git)'
```

- `/review [prompt|path|pattern]` targets staged/unstaged changes; a dedicated built-in `code-review` agent is tuned for high signal-to-noise
- Honors `AGENTS.md`, `.github/copilot-instructions.md`, and path-scoped instruction files -- the same tuning as PR-side reviews applies
- Requires a paid Copilot plan (not Free); auth via GitHub login or `COPILOT_GITHUB_TOKEN`

**Caveats vs CodeRabbit CLI**: output is prose, not structured findings (`--output-format json` is event framing only); exit codes are undocumented; `-p` mode has known reliability rough edges. Prefer CodeRabbit CLI for agent-driven fix loops; use Copilot CLI when the project standardizes on Copilot.

## Key Gotchas

1. **Ruleset detection can 403** -- rulesets need a public repo or GitHub Pro/org plan; on Free private repos the API refuses, yet personal-setting auto-review can still fire. Treat 403 as "unknown, observe" -- never as "disabled".
2. **The personal auto-review setting is invisible** -- no API reads it. An unrequested Copilot review appearing on a fresh PR is the only tell.
3. **A ruleset rule overrides a personal disable** -- users cannot opt out of ruleset-mandated reviews from their own settings.
4. **`review_on_push` multiplies cost** -- every push re-bills a full review. Off = one auto review per PR, further rounds by deliberate re-request.
5. **Budgets may not unblock legacy overage** -- an AI-credits budget does not obviously gate premium-request overage, and entitlement sync lags; exhausted-quota reviews fail with the same opaque "Copilot encountered an error" comment as transient failures.
6. **The generic failure comment can mask a hard weekly rate limit** -- a model-level cap independent of budgets and remaining allowance. Only the review run's Actions log names it (with the reset date); re-requesting before the reset burns Actions minutes for a guaranteed failure. Check the log first (`bot_fail_diag`, `bot-review-loop.md`) and report the reset date.
7. **Effort level and runner type are UI-only** -- do not look for an API; tell the user where the setting lives.
