---
name: bun-cli
description: >-
  Bun CLI reference for package management, script running, testing, bundling,
  and compilation. Covers bun install/add/remove/update, bun run, bun test, bun build, bunx,
  bun patch, bun audit fix, bun dedupe, bun prune, bun why, bun pm, bun repl,
  bunfig.toml, bun.lock, workspace catalogs, isolated installs and the global virtual store,
  zero-config frontend dev, parallel/sequential execution, test sharding and parallelism,
  compile-to-browser, and replacing npm/npx/yarn/pnpm
  with bun equivalents. Use for package management, lockfile issues, test runner config,
  bundler setup, or frontend dev server
  Not for Bun runtime APIs (Bun.file(), Bun.$(), Bun.sql()) -- use bun-api skill
---

# Bun CLI

Bun is an all-in-one JavaScript/TypeScript runtime, package manager, bundler, and test runner. Bun runs TypeScript natively — `bun file.ts` directly, no compile step, no `tsc`, no `ts-node`. Always use `bun` instead of `node`, `npm`, `npx`, `yarn`, or `pnpm` in Bun projects.

**Verified against Bun v1.4.0** (2026-08-20). Features are tagged with the version that
introduced them (`v1.4+`). Where 1.4 changed existing behavior, both behaviors are stated so
this skill stays correct on 1.3.x projects -- run `bun --version` before relying on a
version-tagged item.

## Check the Version, Then Check Bun's Docs

`bun <command> --help` is authoritative for flags and always matches the installed binary --
prefer it over any table here when the two disagree.

For everything beyond flags, Bun ships its full docs inside `bun-types`, version-matched to
the runtime, in any project that has `bun-types` or `@types/bun` installed:

```text
node_modules/bun-types/docs/pm/**/*.mdx        # install, lockfile, workspaces, catalogs, linkers
node_modules/bun-types/docs/test/**/*.mdx      # test runner
node_modules/bun-types/docs/bundler/**/*.mdx   # bun build, executables, loaders, plugins
node_modules/bun-types/docs/runtime/bunfig.mdx
node_modules/bun-types/CLAUDE.md             # Bun's own agent rules
```

1. **Verify the docs match the runtime.** Compare `bun --version` to
   `node_modules/bun-types/package.json`. `bun init` installs `@types/bun@latest`, which lags
   behind releases -- correct it with `bun add -d bun-types@<runtime-version>`.
2. **Open by explicit path.** With the global virtual store enabled, `node_modules/bun-types`
   is a symlink, so `find node_modules -name '*.mdx'` and `rg <pattern> node_modules` return
   nothing. `find node_modules/bun-types/docs -name '*.mdx'` works.
3. **Never edit under `node_modules/`.** Under the global store every project on the machine
   shares the same inode. Use `bun patch`.
4. **Not installed?** The same path works online: `docs/pm/lockfile.mdx` is
   `https://bun.com/docs/pm/lockfile`.

| Task | Doc path (under `node_modules/bun-types/docs/`) |
|---|---|
| Install behavior, lockfile, linkers | `pm/cli/install.mdx`, `pm/lockfile.mdx`, `pm/isolated-installs.mdx`, `pm/global-store.mdx` |
| Workspaces, catalogs, filters, overrides | `pm/workspaces.mdx`, `pm/catalogs.mdx`, `pm/filter.mdx`, `pm/overrides.mdx` |
| Registries, auth, lifecycle scripts | `pm/scopes-registries.mdx`, `pm/npmrc.mdx`, `pm/lifecycle.mdx` |
| Test runner | `test/index.mdx`, `test/discovery.mdx`, `test/configuration.mdx`, `test/code-coverage.mdx`, `test/reporters.mdx`, `test/mocks.mdx` |
| Bundler, executables, loaders | `bundler/index.mdx`, `bundler/executables.mdx`, `bundler/loaders.mdx`, `bundler/plugins.mdx` |
| Config | `runtime/bunfig.mdx` |
| Migrating from npm/yarn/pnpm | `guides/install/from-npm-install-to-bun-install.mdx` |

## Detecting Bun Projects

A project uses Bun if any of these are present:
- `bun.lock` or `bun.lockb` in the project root
- `bunfig.toml` in the project root
- `bun` field in `package.json` (e.g., `"bun": { "install": { ... } }`)
- Package manager field: `"packageManager": "bun@..."`
- `[run] bun = true` in `bunfig.toml` (forces Bun runtime for all scripts)

## Critical Rule

**In a Bun project, ALWAYS use `bun` for everything.** Never fall back to `node`, `npm`, `npx`, `yarn`, or `pnpm`. This avoids compatibility issues, unnecessary retries, and cryptic errors from Node.js/npm not understanding Bun-specific features (workspace protocol, lockfile format, trustedDependencies, etc.).

- Run files: `bun file.ts` (not `node file.ts`)
- Run scripts: `bun run dev` (not `npm run dev`)
- Execute binaries: `bunx tool` (not `npx tool`)
- Install packages: `bun add pkg` (not `npm install pkg`)
- Run tests: `bun test` (not `npx jest` or `node --test`)

## Read-Only Commands (safe, no side effects)

| Command | Purpose |
|---------|---------|
| `bun --version` | Runtime version |
| `bun info <pkg>` | Package metadata, available versions |
| `bun info <pkg> versions` | List all published versions |
| `bun pm ls` | List installed packages (alias: `bun list`) |
| `bun pm ls --all` | List all (including transitive) |
| `bun pm ls --trusted` | List only packages allowed to run lifecycle scripts (v1.4+) |
| `bun pm hash` | Print lockfile hash |
| `bun pm cache` | Show cache directory |
| `bun pm bin` | Print the bin directory (`-g` for global) |
| `bun pm licenses` | List dependencies grouped by license (v1.4+; `--json`, `--prod`) |
| `bun pm diff <pkg>` | Diff two versions of a package (v1.4+) |
| `bun pm untrusted` | List packages whose lifecycle scripts were blocked |
| `bun why <pkg>` | Explain why a package is installed |
| `bun outdated` | Check for outdated dependencies |
| `bun audit` | Security vulnerability audit |
| `bun dedupe --check` | Exit 1 if the lockfile has removable duplicates (v1.4+) |
| `bun prune --dry-run` | Show what would be removed from `node_modules` (v1.4+) |
| `bun test` | Run test suite |
| `bun run lint` | Run linter (project-specific) |
| `bun run check-types` | Type checking (project-specific) |

> **Reference**: See `references/allowlist.md` for copy-paste `Bash(command:*)` patterns for Claude Code / OpenCode settings.

## npm/npx/node to Bun Translation

| npm/npx/node | Bun equivalent |
|---|---|
| `npm install` | `bun install` |
| `npm install pkg` | `bun add pkg` |
| `npm install -D pkg` | `bun add -d pkg` |
| `npm install -g pkg` | `bun add -g pkg` |
| `npm uninstall pkg` | `bun remove pkg` |
| `npm update` | `bun update` |
| `npm run script` | `bun run script` |
| `npx command` | `bunx command` |
| `node file.js` | `bun file.js` |
| `node --watch file.js` | `bun --watch file.js` |
| `npm test` | `bun test` |
| `npm pack` | `bun pm pack` |
| `npm publish` | `bun publish` |
| `npm info pkg` | `bun info pkg` |
| `npm outdated` | `bun outdated` |
| `npm audit` | `bun audit` |
| `npm link` | `bun link` |

### Key Behavioral Differences

- **No `npm run` prefix needed**: `bun run dev` works, but so does `bun dev` (direct script execution)
- **`--bun` flag**: Forces Bun runtime instead of Node.js for scripts that use `node` in their shebang. In `bunfig.toml`, set `[run] bun = true` to make this the default
- **Lockfile**: Bun uses `bun.lock` (text-based, v1.2+) or `bun.lockb` (binary, legacy). Text lockfile is default for new projects
- **Workspace commands**: Use `--filter` flag: `bun --filter 'pkg-name' add dep`
- **Lifecycle scripts**: Bun ignores lifecycle scripts by default for security. Use `trustedDependencies` in package.json to allowlist packages that need postinstall etc.

## Package Management

### Installing Dependencies

```bash
bun install                    # Install all from package.json
bun install --frozen-lockfile  # CI mode: fail if lockfile needs update
bun install --no-save          # Install without updating package.json
bun install --production       # Skip devDependencies
bun install --dry-run          # Show what would be installed
```

### Adding/Removing Packages

```bash
bun add pkg                    # Add to dependencies
bun add pkg@version            # Add specific version
bun add -d pkg                 # Add to devDependencies (--dev)
bun add -D pkg                 # Same as -d
bun add --optional pkg         # Add to optionalDependencies
bun add --peer pkg             # Add to peerDependencies
bun add -g pkg                 # Install globally
bun add --exact pkg            # Pin exact version (no ^)
bun add pkg --filter api       # Add to one workspace from the repo root (v1.4+)
bun add pkg --catalog          # Add to the root catalog, depend on it as "catalog:" (v1.4+)
bun remove pkg                 # Remove package
```

**Changed in 1.4:** `bun add`, `bun remove`, and `bun update` accept `--filter`, and
`bun install <pkg> --filter x` now edits workspace `x` rather than the root. In a workspace
whose default catalog already lists the package, a plain `bun add <pkg>` writes `catalog:`.
`--filter 'web...'` selects `web` and everything it depends on; `'...web'` selects everything
that depends on `web`.

### Updating and Inspecting

```bash
bun update                     # Update all packages within package.json ranges
bun update pkg                 # Update every copy of pkg, transitive ones included (v1.4+)
bun update '@types/*' --latest # Patterns; --latest ignores the declared range
bun update --recursive         # Update every workspace and write each package.json (v1.4+)
bun update --filter 'pkg-*' --filter '!pkg-c'   # Repeatable; ! excludes (v1.4+)
bun outdated                   # Show outdated packages
bun info pkg                   # Show package metadata
bun info pkg versions          # List all available versions
bun why pkg                    # Explain why a package is in the tree
bun pm ls                      # List installed packages (alias: bun list)
bun pm ls --all                # List all (including transitive)
bun pm hash                    # Print lockfile hash
bun pm cache                   # Show cache directory
bun pm cache rm                # Clear cache (including the global virtual store)
```

**Changed in 1.4:** `bun update` re-resolves transitive dependencies, not just the ones named
in `package.json`, and `bun update <name>` **errors with exit 1** when nothing depends on that
name (it used to add the package).

### Maintenance (v1.4+)

```bash
bun audit fix                  # Upgrade vulnerable packages to the lowest safe version
bun audit fix --dry-run        # Preview
bun audit fix --latest         # Also apply fixes your declared ranges exclude
bun dedupe                     # Collapse duplicate versions in bun.lock, then install
bun dedupe --check             # CI gate: exit 1 if duplicates remain
bun prune                      # Drop node_modules entries no longer in bun.lock
bun prune --production         # Also drop devDependencies -- build with them, ship without
bun pm licenses --prod --json  # License inventory
bun pm diff react@18.2.0 19.0.0  # What changed between two versions of a package
```

`bun pm diff` leads with a summary -- files changed, new install scripts, and new imports of
`child_process`, `fs`, `net`, or `vm` -- then the diff, with minified files expanded and
formatting-only changes skipped. Useful for reviewing a dependency bump.

`bun prune --production` fits the Docker build stage:

```dockerfile
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build
RUN bun prune --production
```

### Linking and Patching

```bash
bun link                       # Register current package as linkable
bun link pkg-name              # Link a registered package
bun pm pack                    # Create tarball of package
bun patch pkg                  # Start patching a package
bun patch --commit pkg-dir     # Apply patch
```

### Publishing

```bash
bun publish                    # Publish to npm
bun publish --dry-run          # Preview what would be published
bun publish --tag beta         # Publish with tag
bun publish --access public    # Set access level
```

> **Reference**: See `references/package-management.md` for complete flag details.

## Install Layout: Linker and Global Virtual Store

Two settings decide how `node_modules` is laid out. Both matter more than they look.

### Linker

| | `hoisted` | `isolated` |
|---|---|---|
| Layout | npm-style flat `node_modules` | symlinks into `node_modules/.bun/` (pnpm-style) |
| Phantom dependencies | resolve silently | fail, as they should |
| Default when | `configVersion = 0` (any project), or `configVersion = 1` without workspaces | `configVersion = 1` **and** the project uses workspaces |

The default depends on the lockfile's `configVersion`, not on the Bun version: isolated only
when `configVersion = 1` **and** the project uses workspaces. Lockfiles created before v1.3.2
are treated as `configVersion = 0` and keep hoisted, so upgrading Bun does not change an
existing project's layout. Override with `--linker` or `[install] linker`.

### Global Virtual Store (v1.3.14+, faster in 1.4)

Package files live once in `~/.bun/install/cache/links/` and each project symlinks into them:
warm installs are roughly 7x faster, and `node_modules` drops from hundreds of MB to a few MB
of links. **It is off by default and needs both settings** -- with `linker = "hoisted"` the
`globalStore` flag is silently ignored.

```toml
[install]
linker = "isolated"      # prerequisite
globalStore = true
```

```bash
BUN_INSTALL_GLOBAL_STORE=1 bun install --linker isolated   # per invocation
bun pm cache rm                                            # clears the store too
```

Consequences worth knowing before enabling it:

1. **`node_modules` is mostly symlinks.** `find node_modules -name '*.mdx'` and
   `rg <pattern> node_modules` return nothing, and `du -sh node_modules` reports ~`0B`
   because it measures the symlinks rather than their targets (the real tree is a few MB of
   links; `du -sh -L` shows the target sizes, counted once per link). Address
   files by explicit path, or pass `find -L` (which double-counts through the link layers).
   Tools that scan `node_modules` without following symlinks behave differently -- the same
   caveat as any pnpm-style layout.
2. **Every project shares the same inode.** Editing a file under `node_modules/` changes it
   for every project on the machine. Use `bun patch`.
3. **True phantom dependencies stop resolving.** Packages realpath into the cache, so the
   hidden `node_modules/.bun/node_modules/` layer is no longer on the resolution path from
   inside a package. Fix by declaring the dependency, or set `globalStore = false`.
4. **Some entries stay project-local**: patched packages, `trustedDependencies`, and anything
   depending on a `workspace:`, `file:`, or `link:` dependency. Ineligibility propagates up
   the graph, so a monorepo keeps a good share of the tree local regardless.

> **Reference**: `node_modules/bun-types/docs/pm/global-store.mdx` and `pm/isolated-installs.mdx`.

## Running Scripts and Files

### Direct Execution

```bash
bun file.ts                    # Run TypeScript/JavaScript directly
bun run script-name            # Run package.json script
bun script-name                # Short form (if no conflict with bun commands)
bun --watch file.ts            # Re-run on file changes
bun --hot file.ts              # Hot reload (preserves state)
bun --env-file .env file.ts    # Load env file
bun --env-file .env.local --env-file .env file.ts  # Multiple env files
bun --no-env-file file.ts      # Skip automatic .env loading (CI/prod; `env = false` in bunfig)
bun --no-orphans run dev       # Die with the parent, SIGKILL every descendant on exit
bun repl                       # Native REPL: highlighting, history, completion (v1.3.10+)
bun repl -p '{ a: 1 }'         # Evaluate and print with REPL semantics
bun exec 'ls | wc -l'          # Run a shell script through Bun's shell
bun ./README.md                # Render Markdown in the terminal, no VM started
```

### bunx (npx Replacement)

```bash
bunx command                   # Run package binary (auto-installs if needed)
bunx --bun command             # Force Bun runtime for the command
bunx command@version           # Run specific version
```

### Parallel and Sequential Execution

Replaces `npm-run-all` and `concurrently`. Output is prefixed per script (or `package:script`
under `--filter`), and `pre*`/`post*` hooks stay grouped with their main script.

```bash
bun run --parallel build lint typecheck        # Run all concurrently
bun run --parallel 'build:*'                   # Glob-match script names
bun run --parallel --filter '*' build          # Fan out across every workspace
bun run --parallel --no-exit-on-error --filter '*' test   # Keep going past failures
bun run --sequential clean build deploy        # One at a time, same prefixed output
```

The flag may also precede `run` (`bun --parallel run build lint`); both forms work.

### Workspace-Aware Execution

```bash
bun --filter 'pkg-name' run script    # Run in specific workspace
bun --filter '*' run script           # Run in all workspaces
bun --filter './apps/*' run build     # Run with glob pattern
```

### Script Flags

```bash
bun run --smol file.ts         # Reduce memory usage (sacrifice throughput)
bun run --silent script        # Suppress script name echo
bun run --shell=bun script     # Use Bun's built-in shell (cross-platform, default on Windows)
bun run --shell=system script  # Use system shell (default on macOS/Linux)
```

### Zero-Config Frontend Development

Run HTML files directly as a dev server -- no Vite, Webpack, or any config needed:

```bash
bun ./index.html               # Start dev server, auto-bundles JS/TS/CSS
bun --hot ./index.html         # With hot module replacement
```

Bun automatically transpiles TypeScript, JSX, TSX, and CSS linked from the HTML. Resolves `node_modules` imports in `<script>` tags. Enables HMR and React Fast Refresh.

> **Reference**: See `references/running-and-execution.md` for complete details.

## Testing

Bun includes a built-in test runner compatible with Jest-like syntax.

### Running Tests

```bash
bun test                          # Run all test files
bun test file.test.ts             # Run specific file
bun test foo bar                  # Positional args filter by FILE path, not test name
bun test -t "pattern"             # Filter by TEST NAME (regex); --grep is an alias
bun test --timeout 10000         # Set timeout (ms)
bun test --bail                   # Stop on first failure
bun test --bail 5                 # Stop after 5 failures
bun test --rerun-each 3           # Run each test 3 times
bun test --retry 3                # Default retry count for flaky tests
bun test --only                   # Run only tests marked with .only
bun test --todo                   # Include .todo tests
bun test --only-failures          # Print failures and the summary only
bun test --pass-with-no-tests     # Exit 0 when nothing matches (monorepos)
```

**`--filter` is not a test-name filter.** `-F`/`--filter` selects **workspaces**, and a
positional argument selects **files**. Passing a test name to either matches nothing, and
`bun test` then exits **1** ("no tests found") -- so the mistake fails the run rather than
passing it silently, but it still tests nothing. Use `-t` / `--test-name-pattern` / `--grep`
for test names. `--pass-with-no-tests` turns that exit 1 into 0, which is what would hide it.

### Coverage

```bash
bun test --coverage               # Enable code coverage
bun test --coverage-reporter text # Coverage format: 'text' and/or 'lcov' -- there is no 'json'
bun test --coverage-dir ./cov     # Output directory
```

### Parallelism, Sharding, Isolation

```bash
bun test --parallel               # Files across N worker processes (default: CPU count)
bun test --parallel=4 --isolate   # Explicit worker count
bun test --isolate                # Fresh globalThis per file, same process (Jest/Vitest default)
bun test --shard=1/3              # One CI runner's slice of the files
bun test --changed=main           # Only files your diff reaches (walks the import graph)
bun test --timings=t.json --update-timings   # Record per-file durations
bun test --parallel --shard=1/3 --timings=t.json   # Balance by wall time, not file count
```

- `--parallel` **implies `--isolate`**; `--no-isolate` opts out so each worker reuses one
  global and module registry.
- `--isolate` fixes "passes alone, fails in the suite" by resetting `globalThis` and the module
  registries between files, and closing servers, sockets, watchers, and subprocesses the file
  left open. Transpiled source and bytecode stay cached across globals, so the cost is low.
- Workers expose their 1-indexed slot as `JEST_WORKER_ID` / `BUN_TEST_WORKER_ID`, so Jest
  setups that key a database or port off `JEST_WORKER_ID` work unchanged.
- Coverage and JUnit output are merged across workers; `--bail` stops every worker.
- `--timings` (v1.4+) makes shards equal in **time** rather than file count, and the file is
  written slowest-first so it doubles as a slow-test report.

### Test File Patterns

By default, Bun finds files matching: `*.test.{ts,tsx,js,jsx}`, `*_test.{ts,tsx,js,jsx}`, `*.spec.{ts,tsx,js,jsx}`, `*_spec.{ts,tsx,js,jsx}`, and files in `__tests__/` directories. Exclude paths with `--path-ignore-patterns <glob>` or `test.pathIgnorePatterns` in `bunfig.toml`.

### Snapshot Testing

```bash
bun test --update-snapshots       # Update snapshot files
```

### Watch Mode

```bash
bun test --watch                  # Re-run on file changes
```

> **Reference**: See `references/testing.md` for test API, mocking, lifecycle hooks, and coverage config.

## Bundling and Compilation

### Bundling

```bash
bun build ./src/index.ts --outdir ./dist           # Bundle to directory
bun build ./src/index.ts --outfile ./dist/out.js    # Bundle to single file
bun build ./src/index.ts --target browser           # Target: browser (default), bun, node
bun build ./src/index.ts --format esm               # Format: esm (default), cjs, iife
bun build ./src/index.ts --minify                   # Minify output
bun build ./src/index.ts --sourcemap external        # Sourcemaps: external, inline, linked, none
bun build ./src/index.ts --splitting                # Code splitting (ESM only)
```

### Standalone Executables

```bash
bun build ./src/cli.ts --compile                    # Create self-contained executable
bun build ./src/cli.ts --compile --target bun-linux-x64    # Cross-compile
bun build ./src/cli.ts --compile --minify           # Minified executable
```

Available compilation targets: `bun-linux-x64`, `bun-linux-arm64`, `bun-darwin-x64`, `bun-darwin-arm64`, `bun-windows-x64`.

Browser target (v1.3.10+) -- compile to a self-contained HTML file:
```bash
bun build --compile --target=browser ./app.tsx --outfile ./dist/app.html
```

### Build Options

```bash
bun build ... --external pkg        # Exclude from bundle
bun build ... --define 'KEY=VALUE'  # Define compile-time constants
bun build ... --loader .ext=type    # Custom loaders (js, jsx, ts, tsx, json, css, text, file, base64, dataurl, binary)
bun build ... --entry-naming [dir]/[name].[ext]   # Output naming pattern
bun build ... --public-path /cdn/   # Public path prefix for assets
bun build ... --react-compiler      # React auto-memoization, no Babel/SWC (v1.4+)
bun build ... --metafile meta.json  # esbuild-format build metadata
bun build ... --metafile-md meta.md # Module graph as Markdown, for reading or an LLM (v1.3.8+)
bun build ... --feature=FLAG        # Compile-time flag for `feature()` from bun:bundle
bun build ./src/cli.ts --compile --asset ./public --asset ./templates   # Embed files/dirs (v1.4+)
```

`--asset` (v1.4+) embeds a file or directory into a `--compile` executable keeping original
filenames, so `path.join(import.meta.dir, ...)` resolves the same as on disk. `node:fs` treats
`/$bunfs/` as a real tree, so `readdirSync` (including `recursive`/`withFileTypes`) works
inside the binary -- static file servers that enumerate a directory at startup run unmodified.

**Changed in v1.3.4:** `--compile` binaries no longer auto-load `tsconfig.json` or
`package.json` from the runtime working directory (opt back in with
`--compile-autoload-tsconfig` / `--compile-autoload-package-json`); `.env` and `bunfig.toml`
still auto-load.
`--bytecode` gained ES module support back in **v1.3.9** (`--format=esm`, requires
`--compile`), enabling top-level await,
`import.meta`, dynamic imports, and code splitting -- it previously forced CommonJS.

> **Reference**: See `references/bundling-and-compilation.md` for complete options.

## Project Initialization

```bash
bun init                       # Initialize new project (creates package.json, tsconfig.json, index.ts)
bun init -y                    # Accept defaults, no template picker
bun init --react               # React template (also --react=tailwind, --react=shadcn)
bun init --minimal             # Type definitions only
bun create template-name       # Create from template
bun create next-app my-app     # Example: create Next.js app
```

`bun init` also writes a **`CLAUDE.md` of Bun's own agent rules** into the project root
(copied from `node_modules/bun-types/CLAUDE.md`) — "use `bun` not `node`/`npm`", the API
substitution list, and the HTML-imports frontend pattern. Recommend running it in new Bun
projects: it gives every agent working in the repo the same baseline, and points at the
shipped docs.

**Changed in 1.4:** `bun init` writes `typescript: ^7` (it wrote `^5`, or nothing in the React
templates), and with a non-TTY stdin (CI, a piped spawn) it behaves as `bun init -y` instead
of opening the template picker. `bun update -i` with a non-TTY stdin now exits 1 -- use
`bun update` or `bun outdated`.

## Configuration (bunfig.toml)

Key sections:

```toml
[run]
bun = true                     # Always use Bun runtime (not Node)

[install]
exact = true                   # Pin exact versions by default
peer = false                   # Don't auto-install peer deps
production = false             # Include devDeps
frozenLockfile = false         # Don't fail on lockfile mismatch
globalDir = "~/.bun/install/global"  # Global install location
linker = "isolated"            # "isolated" or "hoisted" -- see gotchas
globalStore = true             # Share package files across projects (requires isolated)

[install.scopes]
"@myorg" = { token = "$NPM_TOKEN", url = "https://npm.pkg.github.com/" }

[test]
coverage = false               # Enable coverage by default
coverageReporter = ["text", "lcov"]
timeout = 5000                 # Default test timeout
pathIgnorePatterns = ["**/e2e/**"]   # Exclude test files by glob

[bundle]
entryPoints = ["./src/index.ts"]
outdir = "./dist"
```

> **Reference**: See `references/configuration.md` for complete bunfig.toml reference.

## Debugging and Profiling

```bash
bun --inspect file.ts              # Start debugger (WebSocket, connect via Chrome DevTools)
bun --inspect-wait file.ts         # Wait for debugger to attach before executing
bun --inspect-brk file.ts         # Break on first line
bun --cpu-prof file.ts             # Generate CPU profile
bun --cpu-prof-md file.ts          # CPU profile as Markdown (v1.3.7+)
bun --heap-prof file.ts            # Generate heap profile
bun --heap-prof-md file.ts         # Heap profile as Markdown (v1.3.7+)
BUN_JSC_logJITCodeForPerf=1 bun file.ts  # Linux perf integration
```

## Environment Variables

```bash
bun --env-file .env file.ts        # Load .env file
bun --env-file .env.local --env-file .env file.ts  # Load multiple (left takes precedence)
```

Bun auto-loads `.env`, `.env.production`, `.env.local`, `.env.production.local` by default based on `NODE_ENV`.

## Built-in Features That Replace External Tools

Bun has many capabilities built in that eliminate the need for external packages or tooling:

### Native TypeScript
Bun runs `.ts`, `.tsx` files directly — no `tsc`, `ts-node`, or `tsx` needed. The transpiler is built into the runtime. Use `bun file.ts` to run any TypeScript file immediately.

### Workspace Catalogs
Bun supports `catalog:` protocol in `package.json` for centralized dependency version management across monorepo workspaces — no need for tools like `syncpack` or `manypkg`:

```json
// Root package.json
{
  "workspaces": ["packages/*"],
  "catalog": {
    "react": "^19.0.0",
    "typescript": "^5.7.0"
  }
}

// packages/app/package.json
{
  "dependencies": {
    "react": "catalog:"
  }
}
```

### Built-in Test Runner
`bun test` is a full Jest-compatible test runner with snapshot testing, mocking, coverage — no need for `jest`, `vitest`, or `mocha`.

### Built-in Bundler
`bun build` replaces `esbuild`, `webpack`, `rollup` for many use cases. Supports code splitting, tree shaking, minification, and standalone executable compilation.

### Built-in SQLite
`import { Database } from 'bun:sqlite'` — zero-dependency SQLite3 with prepared statements and transactions. No need for `better-sqlite3` or `sql.js`.

### Built-in Shell
`Bun.$` tagged template — cross-platform shell execution with automatic escaping. Replaces `execa`, `shelljs`, `zx`.

### Built-in File I/O
`Bun.file()` and `Bun.write()` — fast file operations without importing `fs`. Auto-detects MIME types.

### Built-in Glob
`new Bun.Glob(pattern)` — fast glob matching and file scanning. Replaces `glob`, `fast-glob`, `minimatch`.

### Built-in Password Hashing
`Bun.password.hash()` and `.verify()` with bcrypt and argon2id. Replaces `bcrypt`, `argon2` packages.

### Built-in Compression
`Bun.gzipSync()`, `Bun.deflateSync()`, `Bun.zstdCompressSync()` — no need for `zlib` wrapper packages.

### Built-in Semver
`Bun.semver.satisfies()`, `.order()` — replaces `semver` package.

### Built-in Runtime APIs
For Bun's built-in runtime helpers (`Bun.s3`, `Bun.redis`, `Bun.Archive`, `JSONC`, `JSON5`, `JSONL`, `markdown`, `cron`), see the `bun-api` skill.

### Zero-Config Frontend Dev Server
`bun ./index.html` — serve HTML with auto-bundling of JS/TS/CSS, HMR, and React Fast Refresh. Replaces Vite/Webpack dev server for simple projects.

### ES Decorators
TC39 standard ES decorators supported natively (v1.3.10+) — no `experimentalDecorators` tsconfig needed.

## Key Gotchas

1. **Always use `bun` not `npm`/`node`/`npx`** in Bun projects
2. **Lockfile format**: `bun.lock` (text, v1.2+) is the default for new projects. Legacy `bun.lockb` is binary. Don't mix with `package-lock.json`. **v1.4+** writes `lockfileVersion: 2` (and `3` when nested or version-scoped overrides are used). **Bun 1.3 cannot read version 3** -- check before committing one to a repo whose CI pins an older Bun
3. **trustedDependencies**: Lifecycle scripts (postinstall, etc.) only run for packages listed in `trustedDependencies` in package.json. **v1.3.5+** the default trusted list applies only to packages from the npm registry, so a `file:`/`link:`/`git:`/`github:` dependency named `esbuild` gets no trust from the real esbuild's entry -- list it yourself. **v1.4+** entries match the exact package name rather than a truncated hash
4. **`--bun` flag**: Some tools (e.g., Next.js) use Node.js by default even when run with `bun run`. Use `--bun` or `[run] bun = true` in `bunfig.toml` to force Bun runtime
5. **Workspace protocol**: Use `"workspace:*"` in package.json to reference workspace packages
6. **Global binaries**: Installed with `bun add -g`, located in `~/.bun/bin/`
7. **Node.js compatibility**: Bun implements most Node.js APIs but some edge cases differ. Check https://bun.sh/docs/runtime/nodejs-apis for compatibility
8. **TypeScript**: Bun runs TypeScript natively with no compilation step. Uses its own transpiler (not tsc)
9. **Auto-install**: Bun can auto-install missing packages on import (disabled by default, enable with `[install] auto = true` in bunfig.toml)
10. **`bun run` vs `bun`**: `bun run script` runs a package.json script; `bun file.ts` runs a file directly. `bun script` tries script first, then falls back to file
11. **`--filter` is not a name filter for tests.** It selects workspaces. `bun test -t <regex>` (or `--grep`) filters test names; a positional argument filters file paths. Getting this wrong matches nothing and exits 1 -- unless `--pass-with-no-tests` is set, which turns it into a green run that tested nothing
12. **Coverage reporters are `text` and `lcov` only** -- `--coverage-reporter=json` is rejected
13. **`bunfig.toml` is strict TOML as of v1.4.** Unquoted values, missing newlines between pairs, and integers past `Number.MAX_SAFE_INTEGER` now fail at startup with a `SyntaxError`. `linker = isolated` must be `linker = "isolated"`
14. **Bun invoked as `node` no longer loads `.env` (v1.4+).** Under `bun --bun`, `bunx --bun`, or a `node` symlink, a script calling `node` sees those variables as `undefined`. Pass `--env-file`, matching Node's behavior
15. **The isolated linker is not on by default for existing projects** -- it is chosen by the lockfile's `configVersion`, not the Bun version. See "Install Layout" above
16. **x64 builds are baseline-only (v1.4+).** The `-march=haswell` build is gone; the `-baseline` download URLs and npm packages still exist and contain the same binary, and the `CPU lacks AVX support` warning is removed
17. **`bun feedback` was removed in 1.4**

## References

> - `references/migration-1.4.md` -- what changed between 1.3 and 1.4 for the CLI, package manager, and test runner
> - `references/package-management.md` -- install/add/update/audit/pm flags, workspaces, linkers
> - `references/running-and-execution.md` -- runtime flags, bunx, profiling, watch/hot
> - `references/testing.md` -- test runner API, mocking, coverage, CI shaping
> - `references/bundling-and-compilation.md` -- `bun build`, targets, standalone executables
> - `references/configuration.md` -- full `bunfig.toml` reference
> - `references/allowlist.md` -- copy-paste `Bash(command:*)` permission patterns
