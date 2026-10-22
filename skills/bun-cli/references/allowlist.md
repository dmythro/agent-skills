# Suggested Allowlist Patterns

Copy-paste ready `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config. Covers read-only and commonly safe `bun` commands.

`Bash(x:*)` is a **prefix** match, so it also matches subcommands. Two 1.4 additions make the
obvious broad patterns unsafe:

- **`Bash(bun audit:*)` also allows `bun audit fix`**, which upgrades packages and installs.
- **`Bash(bun pm:*)` also allows** `bun pm trust`, `bun pm cache rm`, `bun pm version`,
  `bun pm pkg set|delete`, and `bun pm migrate`, all of which mutate state -- and, as of v1.4.4,
  `bun pm fetch` and `bun pm cache prune|pack|unpack`, which write to the shared cache or to disk.

Use the exact patterns below for those two.

**`bun check` (v1.4.3+) is not always read-only.** It type checks and never writes files --
unless `package.json` defines a `check` script, which it then runs like `bun run check`. It
therefore sits with the script patterns below, not the inspection list. `bun --check <file>`
runs the file after checking it, so it is never read-only.

**`bun fmt` (v1.4.4+) writes files, and only one form of it is read-only.** `--check` writes
nothing, but a `fmt` script in `package.json` takes over `bun fmt --check` the way a `check`
script takes over `bun check`, and `-w` / `--write` overrides `--check` wherever it appears -- so a
prefix pattern such as `Bash(bun fmt --check:*)` would also allow `bun fmt --check --write`. Allow
only the exact `Bash(bun fmt --no-script --check)`.

**`bun install --dry-run` ran the project's lifecycle scripts before v1.4.4.** On 1.4.4 and later
it writes nothing and runs nothing; on an older Bun it is not a read-only probe.

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
"Bash(bun prune --check:*)",
"Bash(bun install --dry-run:*)",
"Bash(bun fmt --no-script --check)"
```

`bun prune --check` and `bun fmt --no-script --check` are v1.4.4+; `bun install --dry-run` is
read-only from v1.4.4 on (see above).

## Script Execution (Project-Specific)

These run project-defined scripts — safe if your scripts are read-only:

```json
"Bash(bun run:*)",
"Bash(bun test:*)",
"Bash(bun check:*)"
```

Note: `bun run` covers all package.json scripts including `dev`, `build`, `lint`, `check-types`, etc. Only allowlist this if you trust your project's scripts. `bun check` belongs here because a `check` script takes precedence over the type checker.

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
      "Bash(bun prune --check:*)",
      "Bash(bun fmt --no-script --check)",
      "Bash(bun run:*)",
      "Bash(bun test:*)",
      "Bash(bun check:*)"
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
- `bun pm cache rm` — clears the shared cache, the global virtual store, and cached `bunx` packages for every project
- `bun pm fetch` / `bun pm cache prune|pack|unpack` — fill or prune the shared cache, or write a pack file (v1.4.4+)
- `bun fmt` — rewrites files in place; a `fmt` script runs instead when `package.json` has one (v1.4.4+)
- `bun pm version` / `bun pm pkg set|delete|fix` — edits package.json and can create a git tag
- `bun pm migrate` — writes a new lockfile
- `bun build` — writes output files
- `bun publish` — publishes to npm
- `bun link` / `bun patch` — modifies local state
- `bun upgrade` — replaces the installed Bun binary
