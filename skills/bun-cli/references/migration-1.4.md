# Bun 1.3 to 1.4: CLI, Package Manager, and Test Runner Changes

**What changed under existing projects.** Bun's shipped docs describe the current state only --
this file is for upgrading a repo or a CI pipeline written against 1.3.x, plus the 1.4.1,
1.4.2, and 1.4.3 changes at the end. Runtime and API changes live in the `bun-api` skill's
`references/migration-1.4.md`.

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
  `Bun.build()`, and `--windows-*` executable metadata flags.
- Landed earlier in the 1.3 line but easy to miss: `--bytecode` gained ES module support in
  **v1.3.9** (`--bytecode --format=esm`, requires `--compile`).

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

## 1.4.1

### Package Manager

- **`bun install --offline`** (`[install] offline = true`) makes no network requests and fails
  naming the first uncached package. **`--prefer-offline`** (`[install] prefer = "offline"`)
  uses cached manifests regardless of age and downloads only what is missing.
- **Self-contained workspaces**: `"workspaces": { "packages": [...], "selfContained": [...] }`
  in the root `package.json`, or Yarn's `"installConfig": { "hoistingLimits": "workspaces" }`
  in the workspace, give that workspace a complete, copy-based `node_modules` under the hoisted
  linker. Not recorded in the lockfile; no effect under the isolated linker.
- A no-op install in a monorepo with a shared dependency whose peer is unprovided dropped from
  about 13 s to 20 ms. `bun dedupe`, `bun prune --production`, and `bun pm licenses` print
  clearer output. `bun pm pkg set`, `bun pm version`, `bun pack`, and `bun publish` no longer
  rewrite large integers in exponent form.

### Bundler

- **Code splitting emits fewer, smaller chunks.** Code shared between an entry and its lazy
  routes stays in the entry's chunk (a 40-route test: 219 files to 151), and bindings shared
  between chunks get one bundle-wide name (about 19% smaller). Snapshot tests of build output
  will change.
- **`<link rel="modulepreload">` is emitted by default** with `--splitting --target browser`,
  and each `import()` preloads its transitive chunks. `--no-module-preload` /
  `modulePreload: false` restores plain `import()` calls.
- **`--min-chunk-size=N`** (`minChunkSize`) folds side-effect-free chunks smaller than N source
  bytes into the chunk that loads them. Default `0` (off); `16384` is a good browser value.
- **`require()` of an ES module is a chunk boundary** with `--splitting --target bun`; the call
  stays synchronous. `--no-split-require` / `splitRequire: false` keeps the old inlined output.
- **`export * as` and dynamic `import()` tree-shake.** Unused exports behind namespace
  re-exports and `const { x } = await import(...)` are dropped: `zod` 4.5 shrinks from 375 KB to
  77 KB minified, `effect` 3.22 from 369 KB to 164 KB. A namespace that escapes (passed to a
  function, spread, `Object.keys(ns)`, computed access) keeps everything.
- **Namespace object setters are deprecated.** Bundled `import * as ns` / `export * as ns`
  objects accept `ns.x = 1` silently today and will become getter-only. Opt in now with
  `--no-deprecated-namespace-object-setters` / `deprecatedNamespaceObjectSetters: false`.
- **Nested classes and functions keep their names.** A class `Model` returned from a factory
  reported `.name === "Model2"` when a top-level `Model` existed; it is now `"Model"`.
  `--minify` output is unchanged.
- **Default imports of CommonJS compile to a plain variable** when every export is visible
  (`React.createElement` becomes `$createElement`), including through
  `module.exports = require("./impl")` redirects.
- **1.4.0 regressions fixed:** a default import of a CommonJS `.js`/`.ts` file returned all of
  `module.exports` instead of matching `bun run`; `onResolve` returning `undefined` dropped code
  from `sideEffects: false` packages; unused classes with a computed key (`[TypeId]`) were not
  tree-shaken; `--compile` without `--outfile`/`--outdir` wrote next to the entrypoint instead
  of the working directory; `--compile` from a WSL2 `/mnt/c` path failed with `EACCES`.

### Executables (`--compile`)

- **Bytecode is about 3x the source size instead of 9x**, and compiled executables start about
  20% faster. `--bytecode-depth=N` (`bytecodeDepth`) compiles only the top N nesting levels
  ahead of time; `0` is top-level code only, and functions past the limit compile on first call.
- **Cross-compiling with `--bytecode` works for every target**, including
  `--target=bun-windows-x64` from macOS or Linux. The cache format is identical on all
  platforms, so builds are byte-identical.
- Text imports (`with { type: "text" }`) are embedded once, pre-encoded: a 3 MB text import
  cost about 12 MB of peak memory and now costs about 1 MB.
- `import addon from "./x.node"` no longer throws `__require is not defined` in ESM bundles.
- Compiled executables ignore `NODE_COMPILE_CACHE`.

### Runtime Flags

- **`--env-file` reads pipes, FIFOs, and `/dev/stdin`**: `echo A=1 | bun --env-file=/dev/stdin app.ts`,
  or `bun --env-file=<(op inject -i .env.tpl) app.ts` with a secret-manager CLI that prints
  dotenv lines. The default `.env*` lookup still skips non-regular files.
- **`--no-ffi-cc`** disables `cc()` from `bun:ffi` (`ERR_FFI_CC_DISABLED`); `--no-addons`
  disables it too. Workers inherit both; bake it into an executable with
  `--compile-exec-argv="--no-ffi-cc"`.
- `bun run --filter`, `--workspaces`, `--parallel`, and `--sequential` honor an auto-discovered
  `bunfig.toml`; before 1.4.1 they required `--config`.
- `bun run` re-transpiles a file of 4 KiB or more after `[define]` or `--drop` changes; it
  reused a stale cache before.

### Test Runner

- **`--isolate` no longer leaks between files** that use `mock()`, `spyOn()`, `mock.module()`,
  or `Bun.plugin()`: 80 such files went from 1.6 GB to 215 MB. `process.env.TZ`, proxy, and
  TLS settings no longer carry into later files.
- `jest.useFakeTimers({ now })` and `setSystemTime()` move `performance.timeOrigin` together
  with `Date.now()`.
- Piped stdout (`bun test | pbcopy`) no longer receives ANSI colors when only stderr is a TTY.

## 1.4.2

A bugfix release; no new flags or bunfig keys. Two 1.4.1 regressions are fixed -- upgrade
rather than work around them:

### Bundler

- **`bun build` renamed a nested `var` onto a `let` in the same block** (1.4.1 only). A
  function with `{ let exports2 = {}; var exports = exports2 }` in an inner block came out as
  `var exports2 = exports2`, and the bundle failed to load with
  `SyntaxError: Cannot declare a var variable that shadows a let/const/class variable`. Any
  build importing Elysia hit it. The same renamer bug could also give a `let` the name of a
  function parameter or `catch` binding, which loads but computes wrong values -- so a 1.4.1
  bundle that loads is not proof it is correct. Verified fixed on 1.4.2: the inner binding is
  renamed instead.

### Package Manager

- **`bun install` and `bun add` could panic with `range end index out of range`** when a
  `bun.lockb` or a cached registry manifest stored a package-name hash that did not match the
  name. Fixed in 1.4.2; on an older runtime, delete the cached manifest (`bun pm cache rm`) or
  migrate the binary lockfile to `bun.lock`.

## 1.4.3

`bun check` and `--check` on run, test, and build (see `type-checking.md`), profile-guided
bytecode layout, and an enforced `--disallow-code-generation-from-strings`. Verified on the
1.4.3 runtime except items marked (release notes).

### New Commands and Flags

- `bun check` (`-p`, `-b`, `--pretty` / `--no-pretty`, `--all`, `--threads`, `--timing`, and any
  compiler option as a flag), plus `--check` on `bun <file>`, `bun run`, `bun test`,
  `bun build`, and `Bun.build({ check: true })`.
- `bun build`: `--bytecode-order`, `--compile-jit-policy`, `--no-optimize-bytecode`, and the
  `[hash9]` through `[hash13]` naming tokens.
- `bun init` adds `"typecheck": "bun check"`, and the `CLAUDE.md` it copies says `bun check`
  instead of `tsc --noEmit`. It still adds `@types/bun` as `latest`, which resolves to the
  previous release for a day or more after each Bun release -- switch to `bun-types` (SKILL.md,
  "Check the Version").
- `--disallow-code-generation-from-strings` is enforced, and `=strict` extends it to `node:vm`,
  `data:` imports, and plugins returning source text. Bun 1.4.2 accepted the flag and ignored it,
  so a deployment that already passes it starts throwing `EvalError` wherever a dependency
  compiles a string (`running-and-execution.md`).

### Package Manager

- **Installs fetch only what they place.** Without a lockfile, `bun install` and `bun add` skip
  the tarballs of groups dropped by `--production` / `--omit`, of other platforms' packages, and
  of bundled dependencies -- so those never reach the cache. A cache warmed by a `--production`
  install then fails a full `--offline` install (`--offline: "left-pad" is not in the cache`;
  1.4.2 cached everything). Warm `--offline` caches with an install that places the same
  packages, on the same platform.
- `--cafile`, `--ca`, and bunfig `cafile` / `ca` apply to an https registry reached through
  `HTTPS_PROXY`; a self-signed registry behind a proxy failed with
  `DEPTH_ZERO_SELF_SIGNED_CERT` (release notes).
- `bun update <name>` no longer exits 0 and rewrites `package.json` when `<name>` is an optional
  dependency behind an `npm:` alias whose download failed (release notes).

### Bundler

- **`Bun.build({ tsconfig })` is honored.** It was silently ignored in favor of the
  `tsconfig.json` Bun found itself, so a build that passes a custom file now gets its `paths`,
  JSX, and decorator settings -- output can change. `--tsconfig-override` no longer prints
  `Internal error: directory mismatch for directory`.
- **`--jsx-runtime=solid` and bunfig `jsx = "solid"` are errors** (they were accepted).
  tsconfig `jsxImportSource: "solid-js"` now selects the automatic runtime; it emitted
  `React.createElement` calls before (release notes).
- **Chunk names grow on collision.** Different chunks whose 8-character `[hash]` matched failed
  large `--splitting` builds with `Multiple files share the same output path`; both now get
  extra characters until they differ. 1.4.2 printed `[hash13]` literally and failed.
- **Nothing folds into an entry chunk without `[hash]` in its name**; the shared code goes into
  a hashed chunk. Loading such an entry with a query string (`index.js?v=1`) ran it twice -- a
  1.4.1 regression. `--splitting` output snapshots may change (release notes).
- **Metafile:** each split `import()` in `inputs` carries `entryPoint`, the `inputs` key of the
  file it loads. `outputs[...].bytes` and `--metafile-md` counts are accurate.
- **CommonJS default imports stay live.** A module whose default import is used as a value
  (`fn(React)`, `Object.keys(React)`) or has a property deleted keeps its CommonJS wrapper, so
  `Object.defineProperty`, `delete`, and `Object.freeze` on it affect later reads again -- a
  1.4.1 regression (release notes).
- Builds with many entry points use far less memory (800 entries into one 200,000-export file:
  2,121 MB to 276 MB peak), fixing a 1.4.1 regression that could abort on Windows with
  `memory allocation of N bytes failed` (release notes).

### Executables (`--compile`)

- `--compile --bytecode` executables start faster without any flag, and `--bytecode` output is
  3-6% smaller. Profile-guided layout and the JIT policy are opt-in
  (`bundling-and-compilation.md`, "Startup Tuning").
- `--compile --bytecode --format=esm --splitting` builds are reproducible again (release notes).

### Test Runner

- **Negated asymmetric matchers are correct for primitives.**
  `expect(5).toEqual(expect.not.any(Number))` passed on 1.4.2 and fails on 1.4.3, as do
  `expect.resolvesTo.any()` / `expect.rejectsTo.any()` misuses -- a suite that leaned on the bug
  turns red on upgrade.
- `--coverage-reporter` implies `--coverage`; `--coverage-reporter=lcov` alone ran the tests and
  wrote no report.
- `mock.module()` with an async factory for an already-imported module returns a promise that
  resolves once the exports are replaced -- `await` it. It spun at 100% CPU before (release
  notes).
- `bun test --parallel` exits 143 on SIGTERM instead of 130 (release notes).
- Under `--isolate` / `--parallel`, a promise, callback, or `FinalizationRegistry` callback left
  by a finished file no longer runs inside the next file (release notes).

### Platform

- `oven/bun:alpine` is based on Alpine 3.24.
- Bundled root certificates follow NSS 3.128: Entrust Root Certification Authority, ePKI, Atos
  TrustedRoot 2011, and SecureSign Root CA12 are removed (SECOM and Telia added). A registry or
  server chaining only to a removed root fails verification -- pass `--cafile` / `tls.ca`, or
  run with `--use-system-ca` (release notes).

### Regressions Fixed in 1.4.3

Upgrade instead of working around these (release notes):

- **1.4.0, Linux under seccomp** (Docker, Kubernetes): a profile that fails `faccessat2` with
  `EPERM` / `EINVAL` made a hoisted install from a warm cache fail with
  `package was not found in the cache`, and one that denies `prlimit64` made `bun install`,
  `bun build`, and `bun run <script>` panic.
- **1.4.0:** `bun install` rejected a root `file:` package's relative `file:` dependencies that
  point outside the project; `bun pm pack` and `bun publish` panicked with `int cast` on a failed
  tarball write (now `ENOSPC`, exit 1).
- **1.4.1 bundler:** a bundled `const { v } = require("./b.js")` of an ES module inside an import
  cycle saw a later value of `v`; `--minify-syntax` output threw `ReferenceError: m is not
  defined` after `const m = await import(...)`; `--react-compiler` broke a component calling
  `require()` / `import()` in its body; and with `--splitting`, a chunk shared by several entries
  could order an import cycle so a hoisted `var` read `undefined` (opencode hit it on 1.4.1 and
  1.4.2).
- **1.3.14, Linux:** `bun build --compile` run from inside a compiled executable
  (`BUN_BE_BUN=1` or `Bun.build({ compile })`) wrote a binary that crashed at startup.
