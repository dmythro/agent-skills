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

Then find every installed package that declares it as a peer. Read the installed manifests, not the registry.

**bun** (hoisted layout -- manifests sit one or two levels down):

```bash
bun -e 'const {Glob} = await import("bun"); const t = "<pkg>";
  for (const p of ["*/package.json", "@*/*/package.json"])
    for await (const rel of new Glob(p).scan({cwd: "node_modules"})) {
      const m = await Bun.file(`node_modules/${rel}`).json().catch(() => null);
      const r = m?.peerDependencies?.[t];
      if (r) console.log(`${m.name}@${m.version} requires ${t}@${r}`);
    }'
```

**npm, pnpm, yarn (node-modules linker)** -- same idea without a depth assumption:

```bash
find node_modules -name package.json -exec jq -r --arg t "<pkg>" \
  'select(.peerDependencies[$t] // empty)
   | "\(.name)@\(.version) requires \($t)@\(.peerDependencies[$t])"' {} \; 2>/dev/null | sort -u
```

**Do not add `-maxdepth` to that command.** pnpm's isolated layout stores real manifests at `node_modules/.pnpm/<name>@<version>[_<peerhash>]/node_modules/<name>/package.json` -- five levels down. A `-maxdepth 3` written for a flat npm tree returns *nothing* under pnpm and reports "no package requires this peer", which is the same silent-empty failure as a glob that matches no files. Bun's `--linker isolated` mode nests the same way, as does any yarn PnP project (which has no `node_modules` at all -- use `yarn why <pkg> --peers` there).

Cheaper still, when the manager offers it: `pnpm peers check` lists every unmet peer with its requester, and `yarn why <pkg> --peers` reports peer-driven requirements directly.

**Do not use `npm view <pkg> peerDependencies` for this.** Without a version specifier it returns the metadata of the registry's `latest`, which is not what is installed -- so a project on `payload@3.88` reading a newer `payload`'s peers gets the wrong range and the wrong verdict. Installed manifests carry the exact version's peers, cost no network call, and cover transitive packages that a `package.json` scan never reaches. Where the registry is genuinely the right source -- checking what a *future* version would require -- pin the version explicitly: `npm view <pkg>@<version> peerDependencies --json`.

A hit means the package is mandatory regardless of how the application code looks. Peer dependencies are the contract between a plugin and its host: the host declares the peer, the application must supply it, and the plugin never bundles it. Zero imports in application code is the *expected* state, not evidence of redundancy.

Check `peerDependenciesMeta` before deciding -- a peer marked `"optional": true` is genuinely droppable if the feature it enables is unused. Read it from the installed manifest, for the same reason as above:

```bash
jq '.peerDependenciesMeta' node_modules/<host-pkg>/package.json
```

Only reach for the registry when asking what a *different* version declares, and pin that version when you do: `bun info <host-pkg>@<version> peerDependenciesMeta` (or `npm view <host-pkg>@<version> peerDependenciesMeta --json`). An unpinned registry read answers a question about `latest`, which is rarely the question being asked.

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

Use the peer scan from Step 2 above -- it lists every installed package that constrains the target, at the versions actually installed. The narrowest range in that output is the ceiling.

For the whole tree at once rather than one package, run the peer check below.

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

## Whole-Tree Peer Check (bun)

**Only needed in Bun projects.** pnpm ships `pnpm peers check`, yarn reports `YN0060` at install with a hash for `yarn explain peer-requirements`, and npm surfaces `.problems` via `npm ls --all --json`. Bun alone warns once, does not fail, and never names the package that required the range.

This resolves every installed peer range against what is installed, names both sides, and exits non-zero on a problem -- matching what `pnpm peers check` gives pnpm users. Save as `peer-check.ts`:

```typescript
import { Glob } from "bun";

const installed = new Map<string, string>();
const peers: { from: string; dep: string; range: string; optional: boolean }[] = [];

// Bun.Glob has no brace expansion -- scoped packages need their own scan
for (const pattern of ["*/package.json", "@*/*/package.json"]) {
  for await (const rel of new Glob(pattern).scan({ cwd: "node_modules", onlyFiles: true })) {
    const pkg = await Bun.file(`node_modules/${rel}`).json().catch(() => null);
    if (!pkg?.name || !pkg.version) continue;
    installed.set(pkg.name, pkg.version);
    for (const [dep, range] of Object.entries(pkg.peerDependencies ?? {})) {
      peers.push({
        from: `${pkg.name}@${pkg.version}`,
        dep,
        range: range as string,
        optional: pkg.peerDependenciesMeta?.[dep]?.optional === true,
      });
    }
  }
}

let bad = 0;
for (const p of peers) {
  const have = installed.get(p.dep);
  if (!have) {
    if (!p.optional) { console.log(`MISSING  ${p.from} needs ${p.dep}@${p.range}`); bad++; }
    continue;
  }
  if (!Bun.semver.satisfies(have, p.range)) {
    console.log(`CONFLICT ${p.from} needs ${p.dep}@${p.range} -- installed ${have}`);
    bad++;
  }
}
console.log(bad === 0 ? "peers ok" : `${bad} peer problem(s)`);
process.exit(bad === 0 ? 0 : 1);
```

```text
$ bun run peer-check.ts
CONFLICT graphql-tag@2.12.6 needs graphql@^0.9.0 || ... || ^16.0.0 -- installed 17.0.2
1 peer problem(s)
```

`peerDependenciesMeta[dep].optional` is honoured: an optional peer that is simply absent is not a problem, while an optional peer that is installed at a non-satisfying version still is. Hoisted layouts are covered by the top-level scan; under an isolated linker (`--linker isolated`) nested copies exist, so extend the patterns or run it per workspace.

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

**Observed 2026-08-15** against `payload@3.88.0` (published 2026-08-11), `@payloadcms/graphql@3.88.0`, `@payloadcms/next@3.88.0`, `graphql@16.8.0` installed, `graphql@17.0.2` latest. The versions matter: this is a snapshot of one dependency graph, not a permanent fact about Payload. Re-run the commands rather than trusting the conclusion -- once Payload widens the range, the same method returns the opposite verdict, which is the point of recording the method instead of the answer.

```bash
$ bun why graphql
graphql@16.8.0
  └─ myapp (requires 16.8.0)

$ bun info payload@3.88.0 peerDependencies
{ "graphql": "^16.8.1" }

$ bun info @payloadcms/graphql@3.88.0 peerDependencies
{ "graphql": "^16.8.1", "payload": "3.88.0" }

$ bun info @payloadcms/next@3.88.0 peerDependencies
{ "next": ">=15.2.9 <15.3.0 || >=15.3.9 <15.4.0 || >=15.4.11 <15.5.0 || >=16.2.6 <17.0.0",
  "graphql": "^16.8.1", "payload": "3.88.0" }
```

Both questions answered from the peer data, and the answers are the opposite of what an import grep suggests:

- **Why it is here**: all three Payload packages declare `graphql` as a required peer. The CMS builds a GraphQL layer internally whether or not the application queries it. Declared with zero imports is correct, and removing it breaks the install.
- **Can it go to 17 (as of `payload@3.88.0`)**: no. `^16.8.1` caps it at 16.x, and `graphql@17.0.2` is outside it. Installing 17 leaves the tree resolvable but peer-invalid -- `bun install` warns once and continues, so the breakage surfaces at runtime rather than at install. The whole-tree peer check above catches it; `bun outdated` does not.
- **Bonus finding**: the same three calls show the Next.js ceiling is `<17.0.0`, set by the CMS adapter rather than by Next itself, and that the adapter's range excludes several patch windows within 15.x.

The correct report is "required peer, upgrade blocked upstream at `payload@3.88.0`, revisit when Payload widens the range" -- not "unused dependency, safe to remove", which is what every unused-dependency tool would have said. When revisiting, the check is a single command: `bun info payload peerDependencies` against the current release.
