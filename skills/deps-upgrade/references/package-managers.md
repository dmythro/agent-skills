# Package Manager Command Matrix

Checked against bun 1.3.14, npm 11.19.0, pnpm 11.21.0 and yarn 4.18.0 -- by execution where the command runs outside its native project type, and against the tool's own help output otherwise. Where a manager lacks a capability the gap is stated rather than papered over with an approximation.

## Detection

```bash
node -p "require('./package.json').packageManager || 'unset'" 2>/dev/null
ls bun.lock bun.lockb package-lock.json pnpm-lock.yaml yarn.lock deno.lock 2>/dev/null
cat .yarnrc.yml 2>/dev/null    # present => yarn berry (v2+), absent with yarn.lock => yarn classic
```

`packageManager` wins over the lockfile when they disagree -- it is what Corepack and CI enforce, so a lockfile from a different manager is stale state, not the source of truth. Multiple lockfiles in one repo is a finding in itself: report it.

## Interactive Upgrade (User-Driven)

The picking step belongs to the user; these are documented so the delta can be reproduced and described.

| Manager | Command | Notes |
|---------|---------|-------|
| bun     | `bun update -i -r` | `-i` interactive select, `-r` all workspaces, `--latest` ignores ranges, `--filter <glob>` targets workspaces |
| pnpm    | `pnpm update -i -r -L` | `-L`/`--latest` ignores ranges in `package.json` |
| yarn    | `yarn upgrade-interactive` | Built in from v4; v2/v3 need `yarn plugin import interactive-tools`. `yarn up -i` is the equivalent |
| npm     | none | `npm update` respects ranges only and has no interactive mode -- use `bunx taze -I` or `bunx npm-check-updates -i` |

Without `--latest` (bun, pnpm) a non-interactive update stays inside the ranges already declared. Interactive pickers show the latest available version alongside the in-range one, so a major can still be selected deliberately. When the delta contains a major nobody expected, the question to ask is which of the three happened: `--latest`, an interactive pick, or a hand edit to `package.json`.

## Outdated

| Manager | Command | Machine-readable |
|---------|---------|------------------|
| bun     | `bun outdated` | **No.** `--json` is accepted and silently ignored; the table prints regardless |
| npm     | `npm outdated --json` | Yes. Add `--all` for transitive, `--long` for extra fields |
| pnpm    | `pnpm outdated --format json` | Yes. `--long` adds the repo link, `--compatible` limits to in-range, `-r` for workspaces |
| yarn    | none | Berry dropped `yarn outdated`; there is no replacement |

**`npm outdated --json` is the portable answer wherever `node_modules` exists.** It reads `package.json` plus the installed tree and needs no `package-lock.json`, so it works unchanged inside a bun, pnpm or yarn project. The exception is Yarn Plug'n'Play, which has no `node_modules` to read -- there, fall back to `taze` or a lockfile-based comparison.

```json
{
  "graphql": {
    "current": "16.8.0",
    "wanted": "16.8.0",
    "latest": "17.0.2",
    "dependent": "myapp",
    "location": "/path/node_modules/graphql"
  }
}
```

`current` is installed, `wanted` is the newest version the declared range allows, `latest` is the `latest` dist-tag. `current === wanted !== latest` means the range itself is the ceiling -- a pin or a caret below the next major -- not the resolver.

Bun's table carries the same three columns as `Current | Update | Latest`.

## Registry Metadata

| Manager | Command |
|---------|---------|
| npm     | `npm view <pkg> <field...> [--json]` |
| bun     | `bun info <pkg> [<property.path>] [--json]` |
| pnpm    | `pnpm view <pkg> <field...>` (npm-compatible) |
| yarn    | `yarn npm info <pkg> -f <fields> --json` |

**`bun info` requires a `package.json` in the working directory** and fails with `error: Bun could not find a package.json file to install from` otherwise -- including for plain registry reads and version-range queries. Default to `npm view`, which works anywhere.

Fields worth requesting:

```bash
npm view <pkg> repository.url homepage --json     # where the release notes live
npm view <pkg> dist-tags.latest                   # never dist-tags -- canary tags pollute it
npm view <pkg> engines peerDependencies --json    # upgrade blockers
npm view <pkg>@<version> deprecated               # empty output means not deprecated
npm view <pkg> time --json                        # publish date per version
npm view '<pkg>@>=16.8.0 <17' version             # every version in a range, one per line
```

The range form is how Phase 2 enumerates the versions crossed. Quote it -- the shell will otherwise eat the comparison operators.

## Why Is This Installed

| Manager | Command | Notes |
|---------|---------|-------|
| bun     | `bun why <pkg>` | Also `bun pm why`. Supports globs (`bun why "@types/*"`), `--top`, `--depth N` |
| npm     | `npm why <pkg>` | Alias of `npm explain`. `--json` available |
| pnpm    | `pnpm why <pkg> --json` | `--depth N`, `-D` dev only |
| yarn    | `yarn why <pkg> --peers` | `--peers` shows peer-driven requirements, `-R` recursive |

Output names the dependents and the range each one asked for -- that range is the upgrade ceiling.

```text
graphql@16.8.0
  └─ myapp (requires 16.8.0)
```

## Peer Conflicts

```bash
npm install --dry-run --no-audit --no-fund
```

The portable detector, and the only one that prints the full chain:

```text
npm error Conflicting peer dependency: graphql@16.14.2
npm error   peer graphql@"^0.9.0 || ... || ^16.0.0" from graphql-tag@2.12.6
npm error     graphql-tag@"2.12.6" from the root project
```

It writes nothing -- verified in a bun project: no `package-lock.json` is created and `bun.lock` is left untouched.

Manager-native alternatives, all weaker:

| Manager | Behavior |
|---------|----------|
| bun     | `warn: incorrect peer dependency "graphql@17.0.2"` during install. Does not fail, does not name the requiring package, scrolls past in normal output |
| npm     | `npm ls --all --json \| jq '.problems'` -> `["invalid: graphql@17.0.2 /path/node_modules/graphql"]`. **Plain `npm ls` at depth 0 shows the same tree as clean at exit 0** |
| pnpm    | Reports unmet peers during install and in `pnpm why --json` |
| yarn    | `yarn explain peer-requirements [hash]` explains a specific requirement set |

## Audit

| Manager | Command |
|---------|---------|
| bun     | `bun audit --json [--audit-level high] [--ignore <CVE>]` |
| npm     | `npm audit --json [--audit-level=high] [--omit=dev]` |
| pnpm    | `pnpm audit --json [--audit-level <sev>]` |
| yarn    | `yarn npm audit --json [--severity high] [-R] [--no-deprecations]` |

`bun audit --json` emits an object keyed by package name; the version banner goes to stderr, so piping to `jq` is safe:

```bash
bun audit --json | jq -r 'to_entries[] | "\(.key): \(.value | map(.severity) | join(","))"'
```

Advisory objects carry `id`, `url` (the GHSA page), `title`, `severity`, `vulnerable_versions` and `cvss`. `vulnerable_versions` is the field that matters when deciding whether the upgrade in hand actually resolves the advisory.

`bun pm scan` scans the lockfile for vulnerabilities without needing an installed tree.

## Frozen Install and Drift

```bash
bun install --frozen-lockfile          # also --dry-run to check without writing
npm ci --dry-run
pnpm install --frozen-lockfile
yarn install --immutable
```

A failure here means `package.json` and the lockfile disagree -- someone edited a version by hand without reinstalling. Resolve that before validating anything downstream.

## Workspaces

| Manager | Recursive | Filtered |
|---------|-----------|----------|
| bun     | `-r` / `--recursive` | `--filter <pattern>` |
| pnpm    | `-r` / `--recursive` | `--filter <pattern>` |
| yarn    | `yarn workspaces foreach -A` | `yarn workspace <name> <cmd>` |
| npm     | `--workspaces` | `-w <name>` |

Every workspace `package.json` enters the delta. A dependency upgraded in one workspace and not another is a duplicate-major risk, not a cosmetic inconsistency.

## Supply-Chain Cooldowns

All four managers now support declining freshly published versions, which is why a resolver may refuse a version that exists:

| Manager | Flag |
|---------|------|
| bun     | `--minimum-release-age=<seconds>` |
| npm     | `--min-release-age <days>`, `--min-release-age-exclude <pkg\|glob>`, `--before <date>` |
| pnpm    | `minimumReleaseAge` config setting, re-applied to lockfile entries on install unless `--trust-lockfile` is passed; pairs with `--trust-policy` |
| yarn    | Time gate applied by default on `yarn up`; `--no-time-gate` disables it |

When an expected version does not appear in a resolution, check the publish date with `npm view <pkg> time --json` before concluding the registry is wrong.

## Manager-Agnostic Tools

Useful where a manager has a gap, and worth naming explicitly since they detect the manager themselves:

| Tool | Use | Currency |
|------|-----|----------|
| `taze` | Cross-manager outdated and interactive upgrade (`bunx taze -I`), monorepo aware | 21.x, active |
| `npm-check-updates` | Range rewriting in `package.json` (`bunx npm-check-updates -i`) | 23.x, active |
| `knip` | Unused dependencies, exports and files | 6.x, active |
| `depcheck` | Unused dependencies | Last published 2025-08 -- prefer knip |

`knip` and `depcheck` report peer-required packages as unused. Their output is a candidate list for investigation, never a removal list -- see `dependency-audit.md`.

## Deno

The one manager not verified in this matrix. `deno outdated` lists outdated dependencies across `deno.json` and `package.json` imports and applies them with `--update`; `deno info <specifier>` inspects the resolved module graph. Confirm flags against `deno outdated --help` before relying on them. Everything else in this skill -- release notes, migrations, verification gates -- applies unchanged.
