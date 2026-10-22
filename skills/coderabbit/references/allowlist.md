# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json` covering CodeRabbit CLI commands.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches the command prefix with any arguments (including none)
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`)
- Patterns cover both the full binary name and the `cr` alias where included. Only the install script creates `cr`; a Homebrew install has `coderabbit` alone, so there the `cr` lines are inert

## Recommended: Broad Patterns

Read-only or local-only commands -- no code leaves the machine, no quota is consumed:

```json
{
  "permissions": {
    "allow": [
      "Bash(coderabbit auth status)",
      "Bash(coderabbit doctor)",
      "Bash(coderabbit stats:*)",
      "Bash(coderabbit review findings:*)",
      "Bash(coderabbit config validate:*)",
      "Bash(coderabbit usage:*)",
      "Bash(coderabbit pullrequest:*)",
      "Bash(coderabbit code ls:*)",
      "Bash(coderabbit code show:*)",
      "Bash(coderabbit code autopilot status:*)",
      "Bash(coderabbit code autofix status:*)",
      "Bash(cr auth status)",
      "Bash(cr doctor)",
      "Bash(cr stats:*)",
      "Bash(cr review findings:*)",
      "Bash(cr config validate:*)",
      "Bash(cr usage:*)",
      "Bash(cr pullrequest:*)",
      "Bash(cr code ls:*)",
      "Bash(cr code show:*)",
      "Bash(cr code autopilot status:*)",
      "Bash(cr code autofix status:*)"
    ]
  }
}
```

### Why These Are Safe

- `auth status` -- exact match (no `:*`), reports login state only
- `doctor` -- local diagnostics (runtime, storage, git metadata, update policy, connectivity)
- `stats` -- local review statistics
- `review findings` -- replays the **cached** findings from the last review (`:*` covers `--dir <path>`); no new analysis, no upload, no quota. The prefix also matches `--clear`, which only dismisses local cached findings -- no code or quota effect
- `config validate` -- checks a local `.coderabbit.yaml` against the official schema; no review, no code upload
- `usage` -- reads the included-review allowance and billing period (`:*` covers `--agent`); spends nothing. Its aliases `coderabbit --usage` and `coderabbit review --usage` do not match this pattern -- call `coderabbit usage` instead of allowlisting a `review` prefix
- `pullrequest` -- reads CodeRabbit's existing threads and prompts on a PR (`--show-threads`, `--show-prompts`, `--agent`); runs no review, uploads no code
- `code ls`, `code show`, `code autofix status` (v0.9.1; `code autopilot status` on v0.9.0, an alias since) -- read cloud task state only; the docs mark `ls` and `show` safe to rerun freely. Both status patterns are keyed on the action word, so they cannot match `on`, `off` or `resume`

## Review Runs (Opt-In)

Running a review uploads the diff to CodeRabbit's service and consumes an hourly-quota slot. Allowlist these only after confirming both are acceptable for the repos this config applies to (project-level `.claude/settings.json` is the right scope, not global):

```json
{
  "permissions": {
    "allow": [
      "Bash(coderabbit review --committed:*)",
      "Bash(coderabbit review --uncommitted:*)",
      "Bash(coderabbit review --base:*)",
      "Bash(cr review --committed:*)",
      "Bash(cr review --uncommitted:*)",
      "Bash(cr review --base:*)"
    ]
  }
}
```

Keying on an explicit scope flag in first position (`--committed`, `--uncommitted`, or `--base` for full-branch reviews) keeps the canonical invocations (`coderabbit review --committed --base main --agent`, etc.) auto-approved while a bare `coderabbit review` still prompts -- the deliberate-scope nudge. The hidden `--type <scope>` alias still runs a review, so it gets no pattern: it prompts like any unscoped run.

**Caveat**: `:*` matches all remaining arguments, so a review command that appends `--api-key <key>` (or `--use-credits`, which bills the run) would also auto-approve -- prefix patterns cannot exclude a later flag (compare the re-request Caveat in the `git-pr` allowlist). Stored auth (`coderabbit auth login`) makes inline keys unnecessary on a developer machine; in environments where inline keys actually appear (headless/CI), omit these patterns and approve review runs manually.

## Not Included (Manual Approval Required)

- **`coderabbit auth login` / `logout` / `auth org`** -- credential and org-context changes (`auth org --agent` only lists organizations since v0.9.0, but the prefix would also match the interactive switch)
- **`coderabbit update`** -- self-modifying binary update (a Homebrew install updates with `brew upgrade coderabbit` instead)
- **`coderabbit skills`** (`install`, `update` and `uninstall` subcommands since v0.9.1) -- installs, updates or removes CodeRabbit's agent skills for detected agents (writes outside the repo); `--agent` on a subcommand previews a read-only plan and `--confirm <plan-hash>` applies exactly that plan after approval. `uninstall` removes only skills the CLI owns
- **`coderabbit review --use-credits`** -- bills the review as usage-based once the included allowance is spent. The opt-in review prefixes above still match it after a scope flag (see their Caveat): where usage billing is `On demand` and each paid run needs approval, omit those patterns
- **`coderabbit review --remote ...`** -- a server-side review of any installed repo, outside the scope-first prefixes above; approve each run
- **`coderabbit config`** (bare, `--generate`, `apply`) -- writes repository config; only `config validate` is read-only
- **`coderabbit code new`** (v0.9.0) -- starts a billed cloud task
- **`coderabbit code resume`** (v0.9.0) -- watching is read-only, but `-m`, `--steer` and `--answer` send into the task and start billed work, as does a session opened with `--interactive` (v0.9.1); the prefix cannot tell them apart
- **`coderabbit code ask`** (v0.9.0) -- every question is billed
- **`coderabbit code plan`** (v0.9.0) -- read-only alone, but the prefix also matches `--approve`, which records a plan approval
- **`coderabbit code cancel`** (v0.9.0) -- stops the running turn; queued messages do not run after it
- **`coderabbit code goal`** (v0.9.1) -- reading is harmless, but the prefix also matches `--set` and `--clear`, and editing a completed goal restarts it
- **`coderabbit code archive`** (v0.9.1) -- hides the task, makes it read-only and pauses its schedules
- **`coderabbit code push`** (v0.9.0) -- commits and pushes the task's changes, or opens a pull request with `--stacked`
- **`coderabbit code autofix on` / `off` / `resume`** (v0.9.1; `autopilot` on v0.9.0, an alias since) -- Autofix (formerly Autopilot) publishes changes and pushes follow-up fixes on its own
- **`coderabbit code handoff`** (v0.8.1; the v0.8.0 top-level `coderabbit handoff` still works as a hidden alias) -- uploads a session summary, an optional plan, and the discovered session transcript to a new CodeRabbit Cloud task
- **`coderabbit code skills import`** (v0.8.1) -- uploads local agent skills to the cloud Coding Agent library; `--scope organization` publishes them to the whole org, and `--yes` skips the confirmation
- **`coderabbit review --api-key ...`** -- inline credentials: never add a pattern targeting them, and note the opt-in review prefixes above cannot *exclude* them (see their Caveat); skip those patterns where inline keys are a real risk
- **Bare `coderabbit` / `cr`** -- runs an unscoped review of all changes; make scope explicit instead
