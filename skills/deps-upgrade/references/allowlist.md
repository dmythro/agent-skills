# Suggested Allowlist Patterns

Auto-approval patterns for Claude Code `settings.json`. Covers the read-only inspection commands this skill uses across bun, npm, pnpm, yarn and `gh`.

**Take only your project's manager.** In a Bun project the `bun` and `gh` entries are the working set; the npm, pnpm and yarn entries are there for projects using those managers. The one npm entry worth keeping in a Bun project is `Bash(npm outdated:*)`, the single query bun has no machine-readable form for.

**OpenCode**: the same commands work with picomatch format (`"command": "allow"`) in OpenCode config.

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
      "Bash(npm audit --json:*)",
      "Bash(npm query:*)",
      "Bash(npm pkg get:*)",
      "Bash(pnpm outdated:*)",
      "Bash(pnpm view:*)",
      "Bash(pnpm why:*)",
      "Bash(pnpm list:*)",
      "Bash(pnpm audit)",
      "Bash(pnpm audit --json:*)",
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

**Audit is deliberately not a `:*` prefix.** `Bash(npm audit:*)` would also match `npm audit fix`, which rewrites versions to satisfy advisories -- majors included. Prefix patterns match any continuation, so a subcommand that shares the prefix is covered by it. The same reasoning applies to `pnpm audit --fix`. `bun audit` has no fix form and `yarn npm audit` has none either, so both are safe as prefixes.

Audit the rest of your allowlist for this shape: any read-only command with a writing sibling one word deeper needs an exact pattern, not a prefix.

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
