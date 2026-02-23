# Suggested Allowlist Patterns

Copy-paste ready `Bash(command:*)` patterns for Claude Code `settings.json` or OpenCode config. Covers read-only and commonly safe `bun` commands.

## Read-Only / Inspection

```json
"Bash(bun --version:*)",
"Bash(bun info:*)",
"Bash(bun pm ls:*)",
"Bash(bun pm hash:*)",
"Bash(bun pm cache:*)",
"Bash(bun outdated:*)",
"Bash(bun audit:*)"
```

## Script Execution (Project-Specific)

These run project-defined scripts — safe if your scripts are read-only:

```json
"Bash(bun run:*)",
"Bash(bun test:*)"
```

Note: `bun run` covers all package.json scripts including `dev`, `build`, `lint`, `check-types`, etc. Only allowlist this if you trust your project's scripts.

## All Patterns Combined

```json
{
  "permissions": {
    "allow": [
      "Bash(bun --version:*)",
      "Bash(bun info:*)",
      "Bash(bun pm:*)",
      "Bash(bun outdated:*)",
      "Bash(bun audit:*)",
      "Bash(bun run:*)",
      "Bash(bun test:*)"
    ]
  }
}
```

## Not Included (Require Approval)

These modify project state and should go through the approval flow:

- `bun install` — modifies node_modules and lockfile
- `bun add` / `bun remove` — modifies package.json and lockfile
- `bun update` — modifies lockfile
- `bun build` — writes output files
- `bun publish` — publishes to npm
- `bun link` / `bun patch` — modifies local state
