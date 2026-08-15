---
name: deps-upgrade
description: >-
  Validate JavaScript/TypeScript dependency upgrades after an interactive or
  manual update. Establish the real delta from git, detect peer/engine/type
  conflicts, extract breaking changes, migrations and adoptable features from
  release notes, assess held-back or seemingly unused packages, and run
  verification gates. Covers bun, npm, pnpm and yarn. Use when packages were
  just upgraded and need validating, when deciding whether a held-back package
  can move, when asking why a dependency is present at all, or when checking
  release notes for breaking changes and migration steps. Not for picking and
  applying upgrades autonomously, bun CLI usage (bun-cli), or non-JavaScript
  package ecosystems
---

# Dependency Upgrade Validation

**The upgrade is the easy part; proving the project still coheres is the work.** This skill takes over after versions are chosen -- typically `bun update -i -r` or an equivalent interactive pick -- and validates what landed: the real delta from git, peer and engine integrity, breaking changes and migrations across every version crossed, features worth adopting, and verification gates that run the project rather than reading about it. It also answers the inverse question: can a held-back package move, and why is it in `package.json` at all.

## When to Use

- **Validating a batch upgrade** -- "I upgraded a lot of packages, validate everything"
- **Assessing a held-back package** -- "can we upgrade X?", "what blocks it?"
- **Questioning a dependency** -- "why do we even have X, we don't use it"
- **Reading release notes** -- breaking changes, migration steps, new features across the versions crossed
- **Post-upgrade triage** -- type, build or test failures that appeared after an update
- **Pre-upgrade recon** -- what a bump would cost before anyone runs it

## Critical Rules

1. **Establish the delta from git, never from the prompt.** "I upgraded a lot of packages" is a starting point, not an inventory. The lockfile diff is the only complete record -- it carries transitive bumps that `package.json` never shows.
2. **`outdated` reports drift, not compatibility.** A clean `bun outdated` means every dependency sits at the newest version its range allows; it says nothing about whether the tree still resolves, builds or runs. Never report an upgrade as validated on that basis.
3. **Peer ranges of ecosystem packages are the ceiling, not the registry's `latest`.** A framework plugin pinning `next@">=16.2.6 <17.0.0"` caps the framework regardless of what npm calls latest. Read the peers of the packages that wrap the target before proposing a bump.
4. **A peer-required package is required with zero imports.** `graphql` is a mandatory `peerDependency` of `payload`; an app that never writes a query still must declare it. Run `why` and check every installed peer before proposing any removal (see `references/dependency-audit.md`).
5. **Never run a migration, codemod, or version change without explicit approval.** Report the exact command and what it will touch. Schema migrations additionally need an ordering and rollback story before they are worth proposing.
6. **Read notes for every version crossed, not just the target.** A breaking change introduced in `16.9` and unmentioned in `16.14`'s notes is still a breaking change for a project coming from `16.8`.
7. **Verify by running the project.** Typecheck, build and tests are the evidence. A changelog that promises compatibility is not evidence.
8. **Report what you did not check.** Skipped packages, unread notes and untested paths belong in the report; silence reads as coverage.

---

## Package Manager Detection

Check in this order -- the `packageManager` field wins where both exist, since it is what Corepack and CI enforce:

```bash
node -p "require('./package.json').packageManager || 'unset'" 2>/dev/null
ls bun.lock bun.lockb package-lock.json pnpm-lock.yaml yarn.lock deno.lock 2>/dev/null
```

| Lockfile            | Manager      | Interactive upgrade command      |
|---------------------|--------------|----------------------------------|
| `bun.lock` (text) or `bun.lockb` (binary) | bun    | `bun update -i -r`               |
| `pnpm-lock.yaml`    | pnpm         | `pnpm update -i -r -L`           |
| `yarn.lock`         | yarn (check `.yarnrc.yml` for berry) | `yarn upgrade-interactive`  |
| `package-lock.json` | npm          | none built in -- `bunx taze -I` or `bunx npm-check-updates -i` |

Workspaces change the shape of every command: a monorepo needs the recursive or filtered form, and every workspace `package.json` enters the delta.

> **Reference**: See `references/package-managers.md` for the full verified per-manager command matrix.

---

## Phase 0: Establish the Delta

Nothing else is trustworthy until this is exact.

```bash
# Declared changes (root + workspaces)
git diff HEAD -- package.json '**/package.json'

# Resolved changes, including transitive bumps nothing declared
git diff HEAD -- bun.lock            # or pnpm-lock.yaml / yarn.lock / package-lock.json

# Already committed on a branch
git diff origin/main...HEAD -- package.json '**/package.json' bun.lock
```

Classify every change, because the bump type sets how much scrutiny it earns:

| Change                      | Risk    | Treatment                                              |
|-----------------------------|---------|--------------------------------------------------------|
| Major (`1.x` -> `2.x`)      | High    | Full release-note read, migration hunt, feature scan    |
| Minor on `0.x`              | High    | Semver grants no compatibility below 1.0 -- treat as major |
| Prerelease / rc / canary    | High    | Confirm it was intentional; check it is not a stray `--latest` artifact |
| Minor (`1.2` -> `1.5`)      | Medium  | Scan notes for `BREAKING`, `deprecat`, `removed`, `migrat` |
| Patch                       | Low     | Skip notes unless it is a direct runtime dependency or a security fix |
| Transitive only (lockfile)  | Medium  | No notes; confirm the tree still resolves and nothing crossed a major |
| Range widened, version same | Low     | Note it -- the next install can drift without a code change |

**Binary lockfiles produce no readable diff.** For `bun.lockb`, either snapshot the resolved tree before and after with `npm ls --all --json` and diff the snapshots, or convert the project once to the text lockfile that has been the default since Bun 1.2:

```bash
bun install --save-text-lockfile --frozen-lockfile --lockfile-only   # rewrites the lockfile only
rm bun.lockb                                                          # conversion leaves the old one in place
```

`--frozen-lockfile --lockfile-only` keeps the conversion to a re-encoding: no resolution beyond what the lockfile already pins, and no touching `node_modules`. Without them the same command is free to resolve new versions, which changes the delta being measured.

Confirm the working tree matches the lockfile before drawing any conclusion from it:

```bash
bun install --frozen-lockfile --dry-run    # bun and npm (npm ci --dry-run) check without writing
```

pnpm and yarn have no true dry run here: `pnpm install --frozen-lockfile` and `yarn install --immutable` fail fast when the lockfile disagrees with `package.json`, which answers the same question, but they install as a side effect. Run them only where installing is acceptable.

## Phase 1: Static Integrity

Run these before reading a single release note -- they are cheap and they catch the failures that no changelog would have warned about.

**Peer conflicts (the highest-yield check):**

```bash
npm install --dry-run --no-audit --no-fund
```

Portable across managers, writes nothing (no `package-lock.json` is created in a bun or pnpm project), and prints the full chain -- which package demands which range, and who pulled it in. `bun install` only emits `warn: incorrect peer dependency "graphql@17.0.2"`, which does not fail the install and never names the requiring package.

```bash
npm ls --all --json 2>/dev/null | jq '.problems'   # "invalid: <pkg>@<ver>" entries
yarn explain peer-requirements                      # yarn berry equivalent
```

**Engines vs the runtime actually in use:**

```bash
npm view <pkg>@<new-version> engines --json
node --version && bun --version
```

**Deprecations introduced by the bump:**

```bash
npm view <pkg>@<new-version> deprecated
```

**Duplicate majors in the tree** -- two copies of a stateful library (React, GraphQL, an ORM client) is a runtime bug, not a size problem:

```bash
bun why <pkg>          # npm why / pnpm why --json / yarn why --peers
npm ls <pkg> --all
```

**Security posture after the bump:**

```bash
bun audit --json | jq 'to_entries[] | {pkg: .key, advisories: [.value[] | {severity, title, vulnerable_versions}]}'
```

**Type package alignment** -- `@types/*` majors track their runtime package's major. A mismatch surfaces as type errors during Phase 6, not as an install failure, so pair them in the delta table.

> **Reference**: See `references/dependency-audit.md` for the peer-ceiling method, duplicate-major diagnosis, and the removal decision tree.

## Phase 2: Release Notes

Budget this phase by the risk column from Phase 0. Fetching notes for every transitive patch burns context and buries the findings that matter.

```bash
npm view <pkg> repository.url homepage --json      # resolve the repo
npm view '<pkg>@>=16.8.0 <=17.0.2' version         # enumerate every version crossed
gh release view v17.0.0 --repo <owner>/<repo> --json tagName,publishedAt,body
```

Source order, cheapest first: `node_modules/<pkg>/CHANGELOG.md` (free, already on disk, but many packages ship none) -> `gh release view` -> the repo's `CHANGELOG.md` / `UPGRADING.md` / `MIGRATION.md` -> the docs site upgrade guide via WebFetch.

Extract only four things: breaking changes, migration steps, deprecations with their removal version, and additions that could replace code the project already hand-rolls. Everything else is noise.

> **Reference**: See `references/release-notes.md` for repo resolution, the tag-naming fallback ladder, monorepo handling, and `gh` recipes.

## Phase 3: Migrations

Release notes bury migration requirements in prose, and the two kinds fail differently:

- **Code migrations** -- codemods, renamed APIs, moved or reshaped config. Failure is loud and local: the build breaks.
- **Schema and data migrations** -- ORM or CMS model changes needing a generated migration committed and applied in a specific order relative to the deploy. Failure is silent in development and destructive in production.

Both get reported with exact commands, a required-now or optional verdict, and for schema migrations an ordering and rollback note. Neither runs without approval.

> **Reference**: See `references/migrations.md` for detection patterns, per-ecosystem commands, and deploy ordering.

## Phase 4: Held-Back and Suspect Packages

For each package the user did not upgrade, or suspects is unnecessary, answer both questions explicitly:

1. **Why is it here?** Direct and imported / direct because a peer requires it / transitive only and wrongly declared / genuinely unused.
2. **What blocks the upgrade?** Name the package and the peer range that caps it, or state that nothing does and the bump is available.

Never answer the first question from an import grep alone -- that is exactly how a required peer dependency gets deleted.

> **Reference**: See `references/dependency-audit.md` for the full decision tree.

## Phase 5: Adoptable Features

From the notes gathered in Phase 2, surface additions the project could use, ranked by what they delete: features replacing a hand-rolled workaround first, then those removing a dependency, then performance and DX. Check the project actually contains the pattern being replaced before suggesting it -- grep for the old API. These are proposals with an effort estimate, never edits.

## Phase 6: Verification Gates

Cheapest first, stopping at the first failure and attributing it before moving on:

```bash
bun install --frozen-lockfile     # tree resolves as locked
bun run typecheck                 # or: bunx tsc --noEmit
bun run lint
bun run build
bun test
```

Read the actual script names from `package.json` rather than assuming; skip gates the project does not define and say so. Then a runtime smoke check -- boot the dev server, hit one route or entry point that exercises the upgraded packages. Type-clean and build-clean code still fails at runtime on changed initialization, config schemas and adapter APIs.

Map every failure to the package that caused it. "Build fails" is not a finding; "build fails because the config option was renamed in 16.0" is.

## Phase 7: Report

Lead with the table, then the sections that need a decision:

```markdown
| Package | Old -> New | Bump | Risk | Breaking | Migration | Action |
|---------|-----------|------|------|----------|-----------|--------|
| next    | 15.4.2 -> 16.3.1 | major | high | yes | codemod | run codemod |
| graphql | 16.8.0 (held) | -- | -- | -- | -- | blocked by payload peer ^16.8.1 |
```

Then, only where non-empty: **Blocked** (what caps each held package), **Action required** (migrations and code changes, with commands), **Adopt** (optional features with effort), **Remove** (dependencies proven unnecessary, with the evidence), **Security** (advisories resolved or introduced), **Not checked** (skipped packages and untested paths).

Commit the result by what changed, never by the process that produced it -- `fix: update config for renamed option`, not `fix: post-upgrade fixes` (see the `git-commit` skill).

---

## Key Gotchas

1. **`bun outdated` has no JSON output** (through 1.3.x) -- `--json` is silently ignored and the table prints anyway. For machine-readable drift use `npm outdated --json`, which works inside a bun, pnpm or yarn tree because it reads `package.json` and `node_modules` directly.
2. **`bun info` fails outside a project** -- `error: Bun could not find a package.json file to install from`, including for plain registry lookups. `npm view` works from any directory; prefer it for metadata.
3. **`npm install --dry-run` writes nothing** -- verified: no `package-lock.json` appears and an existing `bun.lock` is untouched. It is the portable peer-conflict reporter, and the only one that prints the requiring package.
4. **`npm ls` at depth 0 hides peer invalidity** -- a tree with a violated peer range printed clean at exit 0. Use `npm ls --all --json | jq '.problems'`.
5. **bun writes its banner to stderr and data to stdout** -- `bun audit --json | jq` pipes cleanly; no stripping needed.
6. **`dist-tags` can be polluted** -- some packages carry dozens of canary and experimental tags. Read `dist-tags.latest`, never the first entry.
7. **Release tag naming is inconsistent** -- `v17.0.2`, `17.0.2`, `@scope/pkg@6.0.0` in package-tagged monorepos, and release *names* that carry more than the version (React names tag `v19.2.8` as `19.2.8 (July 21st, 2026)`, so name matching fails where `tagName` matching works). Resolve by ladder, do not guess once and give up.
8. **Canary-heavy repos drown the release list** -- `gh release list --repo vercel/next.js` returns mostly prereleases; pass `--exclude-pre-releases`.
9. **`repository.url` can point at a renamed org** -- React's metadata says `github.com/react/react`. `gh` follows the redirect, so pass it through rather than validating it by hand.
10. **Many packages ship no changelog anywhere** -- not in the tarball, not at the monorepo root. GitHub releases are then the only source, and for a few packages the docs site is.
11. **Yarn berry has no `outdated` command** -- it was not carried over from v1. Use `npm outdated --json` or a manager-agnostic tool.
12. **Transitive bumps never appear in `package.json`** -- a supply-chain incident or a breaking change in a nested dependency is visible only in the lockfile diff.
13. **A same-day publish deserves a second look** -- `npm view <pkg> time --json` dates every version. Every manager now ships a cooldown for this reason (`bun install --minimum-release-age`, `npm --min-release-age`, yarn's time gate), so a version the tool refuses to pick may be held back deliberately, not broken.
14. **`knip` and similar unused-dependency tools flag peer-required packages as unused** -- that is the exact trap Critical Rule 4 exists for. Treat their output as a list of candidates to investigate, never as a removal list.
15. **Lockfile drift outlives the upgrade** -- widening a range without changing the installed version means the next clean install resolves differently. Flag range-only edits even though nothing appears to have changed.

---

> **Reference**: See `references/package-managers.md` for the per-manager command matrix
> **Reference**: See `references/release-notes.md` for release-note and changelog retrieval
> **Reference**: See `references/migrations.md` for codemods and schema migrations
> **Reference**: See `references/dependency-audit.md` for peer ceilings and dependency justification
> **Reference**: See `references/allowlist.md` for auto-approval patterns
