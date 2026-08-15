# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers the read-only inspection commands this skill uses across bun, npm, pnpm, yarn and `gh`.

**Take only your project's manager.** In a Bun project the `bun` and `gh` entries are the working set; the npm, pnpm and yarn entries are for projects using those managers. Allowlisting a manager the project does not use invites the agent to reach for a CLI that may not be installed, and that resolves by its own rules if it is.

**OpenCode uses a different shape** -- the `Bash(command:*)` patterns below do not transfer. Rules are picomatch globs under `permission.bash` in `opencode.json`, mapping a command pattern to `allow`, `ask` or `deny`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": {
      "*": "ask",
      "bun outdated*": "allow",
      "bun info*": "allow",
      "bun why*": "allow",
      "bun audit*": "allow",
      "gh release view*": "allow"
    }
  }
}
```

Keep the catch-all `"*": "ask"` so anything unlisted still prompts. The same containment rule applies as below -- a glob ending in `*` matches any continuation, so write `"pnpm audit --json"` without a trailing `*` rather than letting `--fix` through.

## Pattern Syntax

- `Bash(command:*)` -- colon-star matches a command prefix with any arguments (including none)
- `*` cannot match shell operators (`&&`, `||`, `;`, `|`)
- Prefixes match literally, flag order included: `Bash(npm install --dry-run:*)` does not match `npm install --no-audit --dry-run`

---

## Recommended: Broad Patterns

Every subcommand below reads state without modifying `package.json`, the lockfile, or `node_modules`.

```json
{
  "permissions": {
    "allow": [
      "Bash(bun outdated:*)",
      "Bash(bun info:*)",
      "Bash(bun why:*)",
      "Bash(bun pm why:*)",
      "Bash(bun pm ls:*)",
      "Bash(bun pm pkg get:*)",
      "Bash(bun pm scan:*)",
      "Bash(bun audit:*)",
      "Bash(npm view:*)",
      "Bash(npm outdated:*)",
      "Bash(npm ls:*)",
      "Bash(npm why:*)",
      "Bash(npm explain:*)",
      "Bash(npm audit)",
      "Bash(npm audit --json)",
      "Bash(npm query:*)",
      "Bash(npm pkg get:*)",
      "Bash(pnpm outdated:*)",
      "Bash(pnpm view:*)",
      "Bash(pnpm why:*)",
      "Bash(pnpm list:*)",
      "Bash(pnpm peers check:*)",
      "Bash(pnpm pkg get:*)",
      "Bash(pnpm audit)",
      "Bash(pnpm audit --json)",
      "Bash(yarn why:*)",
      "Bash(yarn info:*)",
      "Bash(yarn npm info:*)",
      "Bash(yarn npm audit:*)",
      "Bash(yarn explain:*)",
      "Bash(gh release list:*)",
      "Bash(gh release view:*)"
    ]
  }
}
```

### Why These Are Safe

- `outdated`, `view`, `info`, `why`, `explain`, `ls`, `list`, `query` -- inspection subcommands with no flag combination that writes
- `npm pkg get` / `bun pm pkg get` -- reads `package.json` fields; the writing forms are `set`, `delete` and `fix`, none of which match a `get` prefix
- `bun pm scan` -- reads the lockfile and queries the advisory database
- `gh release list` / `gh release view` -- read-only; release creation is `gh release create`

**Audit patterns are fully exact, with no trailing `:*` at all.** A prefix pattern matches any continuation, so:

- `Bash(npm audit:*)` also matches `npm audit fix`, which rewrites versions to satisfy advisories, majors included.
- `Bash(pnpm audit --json:*)` also matches `pnpm audit --json --fix` and `--fix=update`. pnpm's `--fix` is a *flag*, so it can follow `--json`; pinning the prefix to `--json` does not contain it.
- `Bash(npm audit --json:*)` is unsafe for the same reason, less obviously: npm accepts the subcommand positionally **after** flags. Verified -- `npm audit --json fix --dry-run` returns an install summary (`"add": [...], "added": 1`), not an advisory report, so it is `npm audit fix` running. Flag-position pinning buys nothing against a positional subcommand.

`bun audit` has no fix form, and `yarn npm audit` has none either, so both remain safe as prefixes.

Audit the rest of your allowlist for all three shapes: a writing sibling one word deeper (`npm audit fix`), a writing *flag* that can follow any prefix you pin (`pnpm audit ... --fix`), and a subcommand accepted positionally after flags (`npm audit --json fix`). The last two are easy to miss precisely because the pattern looks specific. When a read-only command has a writing counterpart anywhere in its grammar, pin the whole string.

---

## Writes Nothing Despite the Name

```json
"Bash(npm install --dry-run --no-audit --no-fund)",
"Bash(npm install --dry-run:*)"
```

`npm install --dry-run` is the portable peer-conflict detector and the only one that prints the requiring package. Verified in a bun project: it creates no `package-lock.json` and leaves `bun.lock` untouched.

Allowlist it only in this exact flag position. The `--dry-run` flag must appear immediately after `install` for the pattern to match, so any other `npm install` invocation still prompts. If that is too fine a distinction for your setup, leave it out -- the cost is one approval per validation run.

---

## Not Included (Manual Approval Required)

These change project state, spend money, or cannot be classified by pattern:

- **`bun update` / `bun install` / `bun add` / `bun remove`** and their npm, pnpm and yarn equivalents -- modify `package.json`, the lockfile and `node_modules`
- **`npm audit fix` / `pnpm audit --fix`** -- changes versions to satisfy advisories, majors included
- **`bunx <codemod>` / `npx <codemod>`** -- codemods rewrite source; `bunx` also executes an arbitrary downloaded package
- **All migration commands** -- `payload migrate`, `drizzle-kit generate|migrate|push`, `prisma migrate *`, `typeorm migration:*`. Schema and data changes need per-invocation approval with the target database confirmed
- **All `gh api` calls** -- read and write cannot be distinguished by pattern
- **`bun run` / `npm run`** -- runs project-defined scripts. Allowlist per project if the scripts are trusted; the verification gates in Phase 6 depend on it
