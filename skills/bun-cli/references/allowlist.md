# Suggested Allowlist Patterns

Copy-paste ready `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config. Covers read-only and commonly safe `bun` commands.

`Bash(x:*)` is a **prefix** match, so it also matches subcommands. Two 1.4 additions make the
obvious broad patterns unsafe:

- **`Bash(bun audit:*)` also allows `bun audit fix`**, which upgrades packages and installs.
- **`Bash(bun pm:*)` also allows** `bun pm trust`, `bun pm cache rm`, `bun pm version`,
  `bun pm pkg set|delete`, and `bun pm migrate`, all of which mutate state.

Use the exact patterns below for those two.

## Read-Only / Inspection

```json
"Bash(bun --version:*)",
"Bash(bun --revision:*)",
"Bash(bun info:*)",
"Bash(bun why:*)",
"Bash(bun outdated:*)",
"Bash(bun audit)",
"Bash(bun audit --json:*)",
"Bash(bun audit --audit-level:*)",
"Bash(bun pm ls:*)",
"Bash(bun pm bin:*)",
"Bash(bun pm hash:*)",
"Bash(bun pm cache)",
"Bash(bun pm licenses:*)",
"Bash(bun pm diff:*)",
"Bash(bun pm untrusted:*)",
"Bash(bun pm default-trusted:*)",
"Bash(bun pm whoami:*)",
"Bash(bun pm why:*)",
"Bash(bun pm pkg get:*)",
"Bash(bun list:*)",
"Bash(bun dedupe --check:*)",
"Bash(bun dedupe --dry-run:*)",
"Bash(bun prune --dry-run:*)",
"Bash(bun install --dry-run:*)"
```

## Script Execution (Project-Specific)

These run project-defined scripts — safe if your scripts are read-only:

```json
"Bash(bun run:*)",
"Bash(bun test:*)"
```

Note: `bun run` covers all package.json scripts including `dev`, `build`, `lint`, `check-types`, etc. Only allowlist this if you trust your project's scripts.

## Recommended: Broad Patterns

```json
{
  "permissions": {
    "allow": [
      "Bash(bun --version:*)",
      "Bash(bun info:*)",
      "Bash(bun why:*)",
      "Bash(bun outdated:*)",
      "Bash(bun audit)",
      "Bash(bun audit --json:*)",
      "Bash(bun audit --audit-level:*)",
      "Bash(bun pm ls:*)",
      "Bash(bun pm hash:*)",
      "Bash(bun pm cache)",
      "Bash(bun pm licenses:*)",
      "Bash(bun pm diff:*)",
      "Bash(bun pm untrusted:*)",
      "Bash(bun pm why:*)",
      "Bash(bun dedupe --check:*)",
      "Bash(bun prune --dry-run:*)",
      "Bash(bun run:*)",
      "Bash(bun test:*)"
    ]
  }
}
```

## Not Included (Require Approval)

These modify project state and should go through the approval flow:

- `bun install` — modifies node_modules and lockfile (`--dry-run` is safe)
- `bun add` / `bun remove` — modifies package.json and lockfile
- `bun update` — modifies lockfile and, with `--recursive`/`--filter`, workspace package.json files
- `bun audit fix` — upgrades packages and installs; may rewrite exact pins (and ranges with `--latest`)
- `bun dedupe` — rewrites `bun.lock` and installs (`--check` / `--dry-run` are safe)
- `bun prune` — deletes from node_modules (`--dry-run` is safe)
- `bun pm trust <names>` / `bun pm trust --all` — enables lifecycle scripts
- `bun pm cache rm` — clears the shared cache and the global virtual store for every project
- `bun pm version` / `bun pm pkg set|delete|fix` — edits package.json and can create a git tag
- `bun pm migrate` — writes a new lockfile
- `bun build` — writes output files
- `bun publish` — publishes to npm
- `bun link` / `bun patch` — modifies local state
- `bun upgrade` — replaces the installed Bun binary
