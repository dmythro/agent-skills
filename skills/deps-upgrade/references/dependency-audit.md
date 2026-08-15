# Dependency Justification and Upgrade Ceilings

Two questions come up constantly during an upgrade review, and both are answered wrongly by the obvious method:

- *Why do we even have this? We never import it.* -- answered wrongly by grepping for imports.
- *Can we upgrade this?* -- answered wrongly by reading the registry's `latest`.

## Why Is This Package Here

Classify every questioned package into exactly one of four states. The classification, not the grep, decides whether it can be removed.

| State | Evidence | Verdict |
|-------|----------|---------|
| **Direct and used** | Declared in `package.json` and referenced in source, config or scripts | Keep |
| **Direct because a peer requires it** | Declared, and an installed package lists it in `peerDependencies` | Keep -- removing it breaks the install |
| **Transitive only, wrongly declared** | Declared, no peer requires it, nothing references it, but other packages depend on it internally | Remove from `package.json`; it stays installed as a transitive |
| **Genuinely unused** | Declared, no peer requires it, nothing references it, nothing depends on it | Remove |

### Step 1: Is it declared, and how

```bash
npm pkg get dependencies devDependencies peerDependencies optionalDependencies
```

A package in `dependencies` that only ever runs at build time is a misplacement worth noting, not a removal.

### Step 2: Does any installed package require it as a peer

This is the step that gets skipped, and it is the one that matters.

```bash
bun why <pkg>                  # npm why / pnpm why --json / yarn why --peers
```

Then find every installed package that declares it as a peer. Read the installed manifests, not the registry:

```bash
find node_modules -maxdepth 3 -name package.json -exec \
  jq -r --arg t "<pkg>" 'select(.peerDependencies[$t] // empty)
    | "\(.name)@\(.version) requires \($t)@\(.peerDependencies[$t])"' {} \; 2>/dev/null
```

**Do not use `npm view <pkg> peerDependencies` for this.** Without a version specifier it returns the metadata of the registry's `latest`, which is not what is installed -- so a project on `payload@3.88` reading a newer `payload`'s peers gets the wrong range and the wrong verdict. Installed manifests carry the exact version's peers, cost no network call, and cover transitive packages that a `package.json` scan never reaches. Where the registry is genuinely the right source -- checking what a *future* version would require -- pin the version explicitly: `npm view <pkg>@<version> peerDependencies --json`.

A hit means the package is mandatory regardless of how the application code looks. Peer dependencies are the contract between a plugin and its host: the host declares the peer, the application must supply it, and the plugin never bundles it. Zero imports in application code is the *expected* state, not evidence of redundancy.

Check `peerDependenciesMeta` before deciding -- a peer marked `"optional": true` is genuinely droppable if the feature it enables is unused:

```bash
npm view <host-pkg> peerDependenciesMeta --json
```

### Step 3: Is it referenced anywhere an import grep would miss

```bash
rg -n "<pkg>" --glob '!node_modules' --glob '!*.lock*' -l | head -20
```

Search the package name as a plain string, not as an import. These references are invisible to import-based analysis and to every unused-dependency tool:

- Config files naming plugins as strings -- ESLint, Tailwind, PostCSS, Babel, Vite, Jest
- Binaries invoked from `package.json` scripts
- Framework convention loading (a plugin discovered by file name or directory)
- Type-only imports, which disappear from the compiled output
- Dynamic `require()` or `import()` built from a variable
- Packages required by a hosting platform or a Docker build rather than the app

### Step 4: Only then, consider removal

Removal is safe when Steps 2 and 3 both come back empty. Verify rather than assert:

```bash
# after removing the declaration
bun install --frozen-lockfile   # then the full gate sequence: typecheck, build, test
```

A removal that passes typecheck and build can still break at runtime -- packages loaded by convention fail on boot, not at compile time. Run the smoke check before reporting a removal as validated.

**Unused-dependency tools do not perform Steps 2 and 3.** `knip` and `depcheck` report peer-required packages as unused, which is exactly how a mandatory peer gets deleted. Their output is a candidate list to run through this tree, never a removal list.

## What Blocks an Upgrade

The registry's `latest` is what exists, not what installs. The ceiling is set by whichever installed package declares the narrowest peer range on the target.

### Find the ceiling

```bash
# Every installed package that constrains the target, at the versions actually installed
find node_modules -maxdepth 3 -name package.json -exec \
  jq -r --arg t "<pkg>" 'select(.peerDependencies[$t] // empty)
    | "\(.name)@\(.version) -> \($t)@\(.peerDependencies[$t])"' {} \; 2>/dev/null | sort -u
```

Then confirm against the resolver, which reports the conflict with its full chain:

```bash
npm install --dry-run --no-audit --no-fund
```

### Report the ceiling, not a vague blocker

Name the package, its range, and what would lift it:

```markdown
graphql 16.8.0 -> 17.x is blocked:
  payload@3.88.0            peer graphql@^16.8.1
  @payloadcms/next@3.88.0   peer graphql@^16.8.1
Lift requires a Payload release widening the range. Track: payloadcms/payload releases.
```

Narrow windows are common in framework plugin ecosystems -- a CMS adapter pinning `next@">=16.2.6 <17.0.0"` caps the framework itself. Check the plugins before proposing a framework major, not after the install fails.

### Ceilings that are not peer ranges

| Ceiling | Detection |
|---------|-----------|
| `engines.node` above the runtime in use | `npm view <pkg>@<ver> engines --json` vs `node --version`, `.nvmrc`, CI config, deploy platform |
| A `@types/*` package with no matching major | `npm view @types/<pkg> versions --json \| jq '.[-5:]'` |
| The package requiring a peer the project cannot supply | Read the new version's own `peerDependencies` |
| An override or resolution pinning it | `npm pkg get overrides resolutions pnpm.overrides` |
| The package deprecated with no successor | `npm view <pkg> deprecated` |

Overrides deserve a specific look: a forced version in `overrides` (npm/bun), `resolutions` (yarn) or `pnpm.overrides` silently wins over every declared range, so a package can appear upgradeable and never move. Overrides also decay -- one added to force a security patch stays after the upstream fix ships, holding the tree back.

## Duplicate Majors

Two copies of a stateful library in one tree is a runtime bug, not a bundle-size issue: React hooks fail across copies, ORM clients hold separate pools, `instanceof` checks fail across module instances.

```bash
npm ls <pkg> --all | grep -E '<pkg>@' | sort -u
bun why <pkg> --depth 3
```

Duplicates after an upgrade usually mean one dependent still requires the old major. Options, in preference order: upgrade the lagging dependent; wait for it to widen its range; use an override with an explicit note on why it is safe and when to remove it. Never add an override silently.

## Type Package Alignment

`@types/*` majors track their runtime package's major. After upgrading a runtime package, check its types partner moved with it:

```bash
npm view @types/<pkg> version
npm ls @types/<pkg>
```

Two other cases: a package that has started shipping its own types makes the `@types/*` entry redundant (`npm view <pkg> types` returns a path), and a `@types/*` package can be deprecated in favor of built-ins. Both are cleanups worth reporting, and both surface only as type errors otherwise.

## Worked Example

A project upgrades its CMS and asks why `graphql` is in `package.json` when the app writes no GraphQL, and whether it can go to 17.

```bash
$ bun why graphql
graphql@16.8.0
  └─ myapp (requires 16.8.0)

$ npm view payload peerDependencies --json
{ "graphql": "^16.8.1" }

$ npm view @payloadcms/next peerDependencies --json
{ "next": ">=16.2.6 <17.0.0", "graphql": "^16.8.1", "payload": "3.88.0" }
```

Both questions answered from the peer data, and the answers are the opposite of what an import grep suggests:

- **Why it is here**: `payload` and `@payloadcms/next` declare `graphql` as a required peer. The CMS builds a GraphQL layer internally whether or not the application queries it. Declared with zero imports is correct.
- **Can it go to 17**: no. `^16.8.1` caps it at 16.x. Installing 17 leaves the tree resolvable but peer-invalid -- `bun install` warns once and continues, so the breakage surfaces at runtime rather than at install.
- **Bonus finding**: the same call shows the Next.js ceiling is `<17.0.0`, set by the CMS adapter rather than by Next itself.

The correct report is "required peer, upgrade blocked upstream, revisit when the CMS widens the range" -- not "unused dependency, safe to remove", which is what every unused-dependency tool would have said.
