# Bun 1.3 to 1.4: CLI, Package Manager, and Test Runner Changes

**What changed under existing projects.** Bun's shipped docs describe the current state only --
this file is for upgrading a repo or a CI pipeline written against 1.3.x. Runtime and API
changes live in the `bun-api` skill's `references/migration-1.4.md`.

Run `bun --version` first. On 1.3.x the pre-1.4 behavior still holds.

## Most Likely to Bite

1. **`bun.lock` is `lockfileVersion: 2`**, and **`3`** when the project uses nested or
   version-scoped overrides. **Bun 1.3 cannot read version 3.** Check any CI job or teammate
   pinned to an older Bun before committing one. Version 2 adds two parse-time checks: an npm
   package resolved to a tarball outside the configured registry must carry an integrity hash,
   and git dependency entries are validated against path traversal. Lockfiles written as v0/v1
   keep loading without those checks; `bun install` migrates them.
2. **`bunfig.toml` is strict TOML** and throws `SyntaxError` at startup on unquoted string
   values, missing newlines between key/value pairs, and integers past
   `Number.MAX_SAFE_INTEGER`. `linker = isolated` must become `linker = "isolated"`.
3. **Bun invoked as `node` no longer loads `.env`.** Under `bun --bun`, `bunx --bun`, or a
   `node` symlink, `.env`, `.env.local`, and `.env.{development,production,test}` are skipped,
   matching Node. `bun file.js` still loads them. A `package.json` script that calls `node`
   under `bun --bun run` sees those variables as `undefined` -- pass `--env-file` to `node`.
4. **Node.js 26 is the compatibility target.** `process.versions.modules` is `147`, so any
   package that picks a prebuilt native addon by `NODE_MODULE_VERSION` needs a `147` build.
5. **`bun update` moves transitive dependencies.** It re-resolves every copy of a package,
   including ones your dependencies depend on. `bun update <name>` now **exits 1** when nothing
   depends on `<name>`; it used to add the package.

## Package Manager

- **`--production`/`--prod` on `update`** means "only update `dependencies` and
  `optionalDependencies`". `-i` updates only the interactive selection.
- **`bunfig.toml` overrides `.npmrc`** for the same key. `.npmrc`-only settings such as
  `//host/:_authToken` still attach to registries declared in `bunfig.toml`.
- **`bun install <pkg> --filter x` edits workspace `x`**, not the root. `bun add y --filter x`
  no longer installs a package named `x`, and `add`/`remove --filter '*'` excludes the root.
- **A plain `bun add x` writes `catalog:`** when the workspace's default catalog already lists
  `x`. `bun audit fix` may rewrite exact pins. `--frozen-lockfile --lockfile-only` writes
  nothing, and override or catalog changes fail a frozen install.
- **One-time lockfile churn** on the first install after upgrading for projects with `catalog:`
  peers, dead `pkg@range` override rows, or nested optional-peer placements that differ from a
  fresh resolve.
- **`trustedDependencies` and `--trust` match the exact package name**, not a truncated hash,
  so a package that only collided with an entry's hash no longer runs lifecycle scripts. Entries
  loaded from a legacy `bun.lockb` still match by hash.
- **`bun install --registry <url>` no longer leaks credentials** to a different host, or when
  the URL downgrades from `https://` to `http://`.
- **`workspace:` ranges are honored only in root and workspace `package.json` files.** Inside a
  downloaded package they now fail to resolve like any other unknown range; they used to create
  a workspace package.
- **`patchedDependencies` cache keys are a SHA-1 of the whole patch file**, not a Wyhash of the
  first 16 KiB. Two projects sharing a cache can no longer pick up each other's patched package.
  Existing `node_modules` are re-patched once.
- **`bun init` writes `typescript: ^7`** (was `^5`, or nothing in the React templates), and with
  a non-TTY stdin behaves as `bun init -y` instead of opening the picker. `bun update -i` with a
  non-TTY stdin exits 1 -- use `bun update` or `bun outdated`.
- **`bun feedback` is removed.**

### New Commands and Flags

`bun audit fix` (`--dry-run`, `-L/--latest`, `--ignore`), `bun dedupe` (`--check`, `--dry-run`,
`--lockfile-only`), `bun prune` (`--production`, `--omit`, `--os`, `--cpu`, `--filter`),
`bun pm licenses`, `bun pm diff`, `bun pm scan`, `bun pm ls --trusted`, `bun why`,
`bun add --catalog`, `bun update --recursive`, repeatable `--filter` (with `!` to exclude) on
`update` and `outdated`, `bun install --minimum-release-age`, `--cpu`, `--os`.

`bun audit`'s severity flag is `--audit-level`, not `--level`.

### Install Layout

- **New monorepos default to the isolated linker** and record `configVersion: 1` in `bun.lock`.
  Existing lockfiles are treated as `configVersion: 0` and keep the hoisted linker, so your
  `node_modules` layout does not change on upgrade. Pin `linker = "hoisted"` to opt out.
- **`install.hoist = false`** (v1.4+) disables the isolated linker's hidden
  `node_modules/.bun/node_modules` fallback, so an undeclared `require()` fails with
  `MODULE_NOT_FOUND` instead of resolving through it. Matches pnpm's `hoist`.
- **`publicHoistPattern` and `hoistPattern`** are read from both `bunfig.toml` and `.npmrc`.
- There is **no `hoistAll` setting** -- use `linker`, `hoist`, `hoistPattern`, and
  `publicHoistPattern`.
- The **global virtual store** (`globalStore = true`, requires `linker = "isolated"`) makes
  `node_modules` a tree of symlinks into `~/.bun/install/cache/links/`. See the SKILL for the
  four consequences; the one that catches agents is that `find`/`rg` over `node_modules` stop
  finding anything.

## Test Runner

- **`jest.resetAllMocks()` / `vi.resetAllMocks()` drop mock implementations**, not just call
  history, matching Jest. A `jest.fn(() => 42)` returns `undefined` afterwards and a `spyOn()`
  spy returns `undefined` until `mockRestore()`. Use `clearAllMocks()` for history only.
- **`expect().toContain()` compares with `===`**, not `Object.is`. `expect([-0]).toContain(0)`
  passes; `expect([NaN]).toContain(NaN)` fails. `toBe()` and `toContainEqual()` are unchanged.
- **`--parallel` implies `--isolate`.** Pass `--no-isolate` to let each worker reuse one global
  and module registry across the files it runs.
- New: `--timings` / `--update-timings`, `--parallel-delay`, `--only-failures`,
  `--pass-with-no-tests`, `--path-ignore-patterns`, `--grep` (alias for `-t`), `--retry N`,
  `--concurrent`, `--randomize`/`--seed`, `--max-concurrency`, `--dots`.
- `bun test --isolate` had several 1.3 stability bugs fixed in 1.4: leaked fake timers,
  module-scope subprocesses outliving the file, `process.chdir()` bleeding into the next file,
  leaked handles pinning the previous global, N-API addons pointing at the old global, and a GC
  crash during the swap between files.

## Bundler

- **`bun build --compile` no longer auto-loads `tsconfig.json` or `package.json`** from the
  runtime working directory. Opt back in with `--compile-autoload-tsconfig` and
  `--compile-autoload-package-json`. `.env` and `bunfig.toml` still auto-load.
- **`"jsx": "react-jsx"` emits the production runtime** (`jsx`/`jsxs` from `<pkg>/jsx-runtime`)
  instead of `jsxDEV`. Set `"jsx": "react-jsxdev"` to keep the development runtime.
- **`useDefineForClassFields: false` is now honored**, moving instance field initializers into
  the constructor and dropping declaration-only fields, as tsc does.
- **`--target browser` honors a package's `browser` field for Node builtins** (`"crypto": false`
  or a remap) before falling back to Bun's polyfill.
- **A bundled `import * as ns` namespace enumerates exports in sorted order** -- update
  snapshots that pinned the old order.
- **`--minify` no longer emits a bare `$` identifier**, which shadowed jQuery in classic scripts.
- **`--metafile` sets a bundled import's `path` to the imported file's `inputs` key**, so
  `metafile.inputs[path]` now matches; it was the raw specifier or an absolute path before.
- **An unresolvable `require()`/`require.resolve()`/`await import()` inside `catch`** bundles as
  a runtime throw instead of failing the build with `Could not resolve`.
- New: `--react-compiler`, `--asset`, `--metafile-md`, `--allow-unresolved`,
  `--reject-unresolved`, `--compile-executable-path`, `optimizeImports` and `files` in
  `Bun.build()`, ESM support for `--bytecode`, and `--windows-*` executable metadata flags.

## Platform

- **x64 releases ship only the baseline build.** The `-march=haswell` build is gone; the
  `-baseline` download URLs and npm packages still exist and contain the same binary, so install
  scripts and `bun upgrade` keep working. The `CPU lacks AVX support` warning is removed.
- Minimum glibc on Linux drops to 2.17 (RHEL/CentOS 7, Amazon Linux 1); documented minimum
  kernel is 3.10.
- Native FreeBSD (x86_64, aarch64) and Windows ARM64 builds; experimental Android builds. These
  are runtime platforms, not `bun build --compile` targets.
- On Linux, Bun no longer sets `prctl(PR_SET_THP_DISABLE)` at startup, so child processes
  inherit the system transparent-huge-pages setting instead of Bun's.
