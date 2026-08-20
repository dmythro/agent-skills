# Package Management Reference

> `bun <command> --help` is authoritative for flags. Concepts and behavior:
> `node_modules/bun-types/docs/pm/**/*.mdx` (`cli/install.mdx`, `lockfile.mdx`,
> `isolated-installs.mdx`, `global-store.mdx`, `workspaces.mdx`, `catalogs.mdx`,
> `overrides.mdx`, `lifecycle.mdx`, `npmrc.mdx`). 1.4 changes: `migration-1.4.md`.

## bun install

Install all dependencies from package.json.

```bash
bun install [flags]
```

| Flag | Description |
|---|---|
| `--frozen-lockfile` | Error if lockfile is out of date (CI) |
| `--no-save` | Don't update package.json or lockfile |
| `--production` | Skip devDependencies |
| `--dry-run` | Show what would be installed |
| `--force` | Reinstall all packages |
| `--verbose` | Verbose logging |
| `--silent` | No logging |
| `--no-progress` | Disable progress bar |
| `--ignore-scripts` | Skip lifecycle scripts |
| `--trust` | Run lifecycle scripts for new dependencies |
| `--concurrent-scripts N` | Max parallel lifecycle scripts |
| `--backend` | Install backend: `clonefile` (macOS default, macOS-only), `hardlink` (Linux/Windows default), `clonefile_each_dir` (macOS-only), `copyfile` (fallback when the others fail), `symlink` (mostly internal for `file:` deps; needs `--preserve-symlinks` at runtime) |
| `--linker hoisted\|isolated` | Dependency layout. The default comes from the lockfile's `configVersion`, not the Bun version: `isolated` only when `configVersion = 1` **and** the project uses workspaces; `hoisted` otherwise, including every pre-v1.3.2 lockfile |
| `--cwd path` | Set working directory |
| `-g, --global` | Install globally |
| `--config path` | Load specific bunfig.toml |
| `--no-verify` | Skip checksum verification |
| `--no-cache` | Disable package cache |
| `--cache-dir path` | Custom cache directory |
| `--omit dev\|optional\|peer` | Exclude a dependency type |
| `--lockfile-only` | Write the lockfile without installing |
| `--minimum-release-age N` | Only install packages published at least N seconds ago (supply-chain guard) |
| `--cpu`, `--os` | Override architecture / OS for optional dependencies (`*` for all) |
| `-a, --analyze` | Install every dependency reachable from the given files, via Bun's bundler |
| `--only-missing` | Only add to package.json if not already present |
| `--network-concurrency N` | Max concurrent network requests (default 48) |

Since v1.3.13, `bun install` streams tarballs to disk during download instead of buffering them in memory, so large installs use dramatically less memory.

### Lockfile Behavior

- `bun.lock` (text, JSONC format) — default since Bun v1.2
- `bun.lockb` (binary) — legacy format
- Convert: `bun install` regenerates lockfile; delete old format first
- `--save-text-lockfile` — force text format
- `--frozen-lockfile` — CI mode, errors if lockfile needs changes

**Versioning.** `bun.lock` carries two independent numbers:

| Field | Meaning |
|---|---|
| `lockfileVersion` | Lockfile format. `1` pre-1.4; **`2`** on v1.4+; **`3`** when nested or version-scoped overrides are used |
| `configVersion` | Which install defaults apply. `1` for lockfiles created on v1.3.2+, `0` for older ones (and for lockfiles migrated from npm/yarn) |

**Bun 1.3 cannot read `lockfileVersion: 3`** — verify no CI job or teammate is pinned to an
older Bun before committing one. Version 2 adds two parse-time checks: an npm package resolved
to a tarball outside the configured registry must carry an integrity hash, and git dependency
entries are validated against path traversal. v0/v1 lockfiles keep loading without
those checks; `bun install` migrates them.

`configVersion` is why upgrading Bun does not silently relayout an existing project: the linker
follows the lockfile, not the binary. `configVersion = 0` (any project) stays hoisted;
`configVersion = 1` is isolated for workspaces and hoisted for single-package projects.

Since v1.3.10, `bun.lock` records a SHA-512 hash for GitHub and tarball dependencies, the way
it always has for npm packages. Existing lockfiles pick the hashes up on the next install.

## bun add

Add one or more packages.

```bash
bun add [flags] pkg [@version] [pkg2...]
```

| Flag | Description |
|---|---|
| `-d, --dev` | Add to devDependencies |
| `--optional` | Add to optionalDependencies |
| `--peer` | Add to peerDependencies |
| `-g, --global` | Install globally |
| `-E, --exact` | Pin exact version (no ^ or ~) |
| `--no-save` | Don't update package.json |
| `--trust` | Run lifecycle scripts for this package |
| `--verbose` | Verbose logging |
| `--cwd path` | Set working directory |
| `-F, --filter pattern` | Add in specific workspace(s) — edits that workspace, not the root (v1.4+) |
| `--catalog[=NAME]` | Add the resolved version to the root catalog and depend on it as `catalog:` (v1.4+) |
| `--only-missing` | Skip packages already in package.json |

In a workspace whose default catalog already lists the package, a plain `bun add <pkg>` writes
`catalog:` rather than a version range (v1.4+).

### Version Specifiers

```bash
bun add pkg                  # Latest
bun add pkg@1.2.3            # Exact
bun add pkg@^1.2.0           # Compatible
bun add pkg@~1.2.0           # Patch-level
bun add pkg@latest           # Latest tag
bun add pkg@next             # Next tag
bun add user/repo            # GitHub
bun add https://example.com/pkg.tgz  # Tarball URL
bun add ./local-pkg          # Local path
bun add file:./local-pkg     # Local path (explicit)
```

## bun remove

Remove one or more packages.

```bash
bun remove pkg [pkg2...] [flags]
```

| Flag | Description |
|---|---|
| `-g, --global` | Remove globally |
| `--cwd path` | Set working directory |
| `--filter pattern` | Remove from specific workspace(s) |

## bun update

Update packages to latest compatible versions.

```bash
bun update [pkg] [flags]
```

| Flag | Description |
|---|---|
| `-L, --latest` | Update to latest, ignoring semver ranges |
| `-F, --filter pattern` | Update in specific workspace(s); repeatable, `!name` excludes (v1.4+) |
| `-r, --recursive` | Update every workspace and write each `package.json` (v1.4+) |
| `-i, --interactive` | Pick packages from a list (exits 1 with a non-TTY stdin) |
| `-d, --dev` | Only devDependencies |
| `--no-optional` | Skip optionalDependencies |
| `-E, --exact` | Write exact versions instead of ranges |
| `-p, --production` | Only `dependencies` and `optionalDependencies` (v1.4+ meaning) |
| `--cwd path` | Set working directory |

```bash
bun update                     # every dependency within its declared range
bun update zod                 # every copy of zod, transitive ones included (v1.4+)
bun update '@types/*' --latest # glob patterns
bun update --recursive --latest
bun update --filter 'pkg-*' --filter '!pkg-c'
```

**Changed in 1.4:** `bun update` re-resolves transitive dependencies, not only what
`package.json` names, and `bun update <name>` **exits 1** when nothing depends on `<name>`
(it used to add the package). With no package names it also rewrites the root `catalog` and
`catalogs` entries, leaving `catalog:` references in workspace `package.json` files in place.

## bun outdated

Show packages with newer versions available.

```bash
bun outdated [pkg] [flags]
```

| Flag | Description |
|---|---|
| `-F, --filter pattern` | Check specific workspace(s); repeatable (v1.4+) |
| `--cwd path` | Set working directory |

Output columns: Package, Current, Update (semver-compatible), Latest, Workspace.

## bun audit

Check for known vulnerabilities.

```bash
bun audit [flags]
```

| Flag | Description |
|---|---|
| `--audit-level` | Minimum severity: `low`, `moderate`, `high`, `critical` (**not** `--level`) |
| `--ignore` | Ignore an advisory by GHSA or numeric ID (repeatable) |
| `--json` | JSON output |
| `--cwd path` | Set working directory |

### bun audit fix (v1.4+)

Upgrades vulnerable packages to the lowest safe version that still satisfies every dependent's
range, then installs. `package.json` changes only when an exact pin has to move.

```bash
bun audit fix                  # apply fixes
bun audit fix --dry-run        # preview
bun audit fix --latest         # also apply fixes your declared ranges exclude, rewriting them
```

Fixes blocked by a dependent's range are reported rather than forced.

## bun dedupe (v1.4+)

Removes duplicate versions from `bun.lock` by re-resolving ranges onto versions already in the
lockfile, then installs. Never edits `package.json`.

```bash
bun dedupe
bun dedupe --check             # CI gate: exit 1 if removable duplicates exist, change nothing
bun dedupe --dry-run           # print what would be removed
bun dedupe --lockfile-only     # rewrite bun.lock without installing
bun dedupe --frozen-lockfile   # fail instead of rewriting
```

## bun prune (v1.4+)

Removes packages from `node_modules` that are no longer in `bun.lock`.

```bash
bun prune
bun prune --production         # also remove packages only devDependencies need (alias --prod)
bun prune --omit optional
bun prune --dry-run
bun prune --os linux --cpu arm64   # prune for another platform
bun prune --filter 'pkg-*'         # only the matching workspaces' node_modules
```

Build with devDependencies and ship without them:

```dockerfile
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build
RUN bun prune --production
```

## bun why

Explains why a package is in the tree.

```bash
bun why react
bun why "@types/*" --depth 2
bun why "*-lodash" --top       # top-level dependents only
```

## bun link

Create/use local package links for development.

```bash
bun link               # Register current package as linkable
bun link pkg-name      # Link a registered package into current project
bun unlink             # Unregister current package
```

## bun patch

Patch installed packages (modifies in node_modules, persists via patchedDependencies).

```bash
bun patch pkg[@version]         # Start patching (extracts to temp dir)
bun patch --commit patch-dir    # Apply and record patch
```

Patches stored in `patches/` directory, recorded in package.json `patchedDependencies`.

## bun pm

Package manager utilities.

```bash
bun pm ls [--all]       # List installed packages (alias: bun list)
bun pm ls --trusted     # Only packages allowed to run lifecycle scripts (v1.4+)
bun pm why <pkg>        # Dependency tree explaining why a package is installed
bun pm bin [-g]         # Print the bin directory
bun pm hash             # Print lockfile hash (for CI caching)
bun pm hash-string      # Print the string used to hash the lockfile
bun pm hash-print       # Print the hash stored in the lockfile
bun pm cache            # Show global cache directory
bun pm cache rm         # Clear global cache (and the global virtual store)
bun pm pack [--dry-run] [--destination dir] [--gzip-level 0-9]  # Create tarball
bun pm licenses         # Group dependencies by license (v1.4+)
bun pm diff <pkg>       # Diff two versions of a package (v1.4+)
bun pm scan             # Scan lockfile packages -- requires a security scanner in bunfig.toml
bun pm version [patch|minor|major|...]  # Bump package.json version and tag
bun pm pkg get|set|delete|fix           # Read/write package.json fields
bun pm whoami           # Current npm username
bun pm default-trusted  # Show default trusted dependencies
bun pm untrusted        # Show packages with blocked scripts
bun pm trust [pkg...] [--all]   # Trust packages (allow lifecycle scripts)
bun pm migrate          # Migrate from npm/yarn/pnpm lockfile without installing
```

`bun pm licenses` takes `--json`, `--prod`, `--dev`, `--long`, and `--filter <pattern>`.

`bun pm diff` (v1.4+) is not listed in `bun pm --help` but works. It leads with a summary --
files changed, new install scripts, and new imports of `child_process`, `fs`, `net`, or `vm` --
then the diff, with minified files expanded and formatting-only changes skipped:

```bash
bun pm diff react                     # lockfile version -> latest
bun pm diff react@18.2.0 19.0.0       # two published versions
bun pm diff ./vendored-pkg pkg@2.1.0  # a folder against a published version
bun pm diff react-dom@18.2.0 18.3.1 '*.min.js'
```

## bun info

Show package metadata from the registry.

```bash
bun info pkg                    # Show package info
bun info pkg versions           # List all published versions
bun info pkg version            # Show latest version
bun info pkg description        # Show description
bun info pkg dependencies       # Show dependencies
```

## bun publish

Publish package to npm registry.

```bash
bun publish [flags]
```

| Flag | Description |
|---|---|
| `--dry-run` | Preview without publishing |
| `--tag name` | Publish with dist-tag (default: latest) |
| `--access public/restricted` | Set package access level |
| `--otp code` | One-time password for 2FA |
| `--auth-type legacy/web` | Auth type |
| `--token token` | Auth token |
| `--registry url` | Custom registry URL |

Since v1.3.14, `bun publish` auto-detects the package `README` and sends its contents as npm registry metadata.

## Isolated Installs (Workspaces)

Isolated installs give each workspace package access to only its own declared dependencies,
preventing the monorepo pitfall where code works locally (a sibling workspace installed the
package) but breaks in production.

**It is the default only for _new_ workspace projects.** The default comes from the lockfile's
`configVersion`, not the Bun version:

| `configVersion` | Uses workspaces? | Default linker |
|---|---|---|
| `1` | yes | `isolated` |
| `1` | no | `hoisted` |
| `0` | either | `hoisted` |

Lockfiles created before v1.3.2 are treated as `configVersion = 0`, so upgrading Bun never
changes an existing project's `node_modules` layout. Migrating from pnpm yields
`configVersion = 1`; from npm or yarn, `0`.

```bash
bun install --linker isolated
bun install --linker hoisted
bun --filter 'my-package' add missing-dep   # fix an undeclared import
```

```toml
[install]
linker = "isolated"        # or "hoisted"; must be quoted -- bunfig.toml is strict TOML in 1.4
```

There is **no `hoistAll` setting.** Control hoisting with these keys, readable from both
`bunfig.toml` and `.npmrc`:

| Key | Effect |
|---|---|
| `linker` | `"isolated"` or `"hoisted"` |
| `publicHoistPattern` | Globs hoisted to the **root** `node_modules` so every workspace resolves them (`["@types*", "*eslint*"]`). Like pnpm's `public-hoist-pattern`. Default `[]` |
| `hoistPattern` | Globs hoisted into the virtual store's fallback `node_modules/.bun/node_modules`. Default is everything (`["*"]`) |
| `hoist` | `false` skips creating that fallback directory entirely, so an undeclared `require()` fails with `MODULE_NOT_FOUND` instead of resolving through it (v1.4+). Takes precedence over `hoistPattern` |

## Global Virtual Store (v1.3.14+, faster in 1.4)

Package files are extracted once into `~/.bun/install/cache/links/` and symlinked into each
project, instead of being copied per project. Warm installs run roughly 7x faster and
`node_modules` shrinks to a few MB of links.

**Off by default, and requires the isolated linker** -- with `linker = "hoisted"` the setting
is silently ignored.

```toml
[install]
linker = "isolated"
globalStore = true
```

```bash
BUN_INSTALL_GLOBAL_STORE=1 bun install --linker isolated
bun pm cache rm      # clears the store along with the package cache
```

Store entries are keyed by `(package, version, resolved dependency closure)`, so two projects
share an entry only when their transitive resolutions match.

**Never globally stored** (these fall back to a per-project `node_modules/.bun/<storepath>/`):
patched packages, packages in `trustedDependencies`, and anything depending on a `workspace:`,
`file:`, or `link:` dependency. Ineligibility propagates up the graph.

### Consequences

1. **`node_modules` is mostly symlinks.** `find node_modules -name '<pattern>'` and
   `rg <pattern> node_modules` return nothing, and `du -sh node_modules` reports ~`0B` --
   it measures the symlinks, not their targets. The tree really is only a few MB of links;
   `du -sh -L` follows them and reports the target sizes. Address
   files by explicit path; `find -L` follows links but double-counts through the layers.
2. **Projects share the same inode.** Editing a file under `node_modules/` changes it for every
   project on the machine. Use `bun patch`.
3. **True phantom dependencies stop resolving**, because packages realpath into the cache and
   the hidden hoisted layer is no longer on their resolution path. Declare the dependency, or
   set `globalStore = false`.
4. **The store grows** as new versions and peer-dependency combinations land. `bun pm cache rm`
   clears it; the next install repopulates only what the project needs.

> **Reference**: `node_modules/bun-types/docs/pm/global-store.mdx`

## Workspace Commands

```bash
bun --filter 'pkg-name' install     # Install in specific workspace
bun --filter 'pkg-name' add dep     # Add dep to specific workspace
bun --filter '*' run build          # Run in all workspaces
bun --filter './apps/*' run dev     # Run with glob pattern
bun --filter '!pkg-name' test       # Exclude specific workspace
bun run --filter 'web...' build     # web and everything it depends on
bun run --filter '...web' build     # everything that depends on web
```

`add`/`remove --filter '*'` no longer includes the root workspace (v1.4+).

## Dependency Overrides

npm's nested form, yarn's `a/b`, and pnpm's `a>b` all work, and an override can be scoped to a
version range (v1.4+). Lockfiles using nested or version-scoped overrides are written as
`lockfileVersion: 3`, which **Bun 1.3 cannot read**.

```json
{
  "overrides": {
    "express": { "qs": "6.13.0" },
    "lodash@<4.17.21": "4.17.21"
  }
}
```

## Lifecycle Script Controls

```json
{
  "trustedDependencies": ["esbuild"],
  "nativeDependencies": ["esbuild", "my-custom-package"],
  "ignoreScripts": ["sharp", "another-package"]
}
```

- `trustedDependencies` — allowlist for lifecycle scripts. Since v1.3.5 Bun's built-in default
  trusted list applies **only to packages from the npm registry**, so a `file:`, `link:`,
  `git:`, or `github:` dependency named `esbuild` inherits nothing from the real esbuild's
  entry. Since v1.4 entries match the exact package name rather than a truncated hash.
- `nativeDependencies` (v1.3.2+) — for packages shipping prebuilt binaries as per-platform
  `optionalDependencies` (esbuild and `@esbuild/darwin-arm64`), Bun links the right binary
  directly instead of running `postinstall`. Disable with
  `BUN_FEATURE_FLAG_DISABLE_NATIVE_DEPENDENCY_LINKER=1`.
- `ignoreScripts` (v1.3.2+) — skip a package's lifecycle scripts entirely, even when it is also
  in `trustedDependencies`. Disable with `BUN_FEATURE_FLAG_DISABLE_IGNORE_SCRIPTS=1`.

> **Reference**: `node_modules/bun-types/docs/pm/lifecycle.mdx` and `pm/overrides.mdx`

### Workspace Protocol in package.json

```json
{
  "dependencies": {
    "@scope/pkg": "workspace:*",
    "@scope/pkg2": "workspace:^1.0.0"
  }
}
```
