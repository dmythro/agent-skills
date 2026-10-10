# Type Checking (bun check)

> `bun check --help` is authoritative for flags. Full guide:
> `node_modules/bun-types/docs/runtime/check.mdx` (v1.4.3+).

`bun check` (v1.4.3+) is a port of typescript-go built into Bun. It reads `tsconfig.json`,
reports the errors `tsc` from TypeScript 7 reports (the exact version is
`process.versions.typescript`, `7.0.2` in Bun 1.4.3), uses every CPU core, and never writes a
file. It replaces `tsc --noEmit`. Keep `tsc` only to emit `.d.ts` files.

## Setup

1. **Depend on `bun-types`, not `@types/bun`.** `@types/bun` lags every Bun release, often by
   a day or more, so `bun check` rejects the new APIs with `TS2339` / `TS2353` until it
   catches up. `bun-types` ships with each release:

   ```bash
   bun remove @types/bun
   bun add -d bun-types@$(bun --version)
   ```

   ```json
   { "compilerOptions": { "types": ["bun-types"] } }
   ```

   Adding `bun-types` next to `@types/bun` is not enough: `@types/bun` keeps its own nested
   copy at its pinned version, and `"types": ["bun"]` loads that one.
2. **Keep a `tsconfig.json`.** Without one, `bun check` uses the options `bun init` writes and
   picks up `@types/bun` only. A project on `bun-types` with no `tsconfig.json` fails with
   `Cannot find name 'Bun'`.
3. **The `typescript` package is not needed.** Editors still run their own TypeScript; to keep
   them in agreement, install the matched version:
   `bun add -d typescript@$(bun -p process.versions.typescript)`.
4. `bun init` (v1.4.3+) adds `"typecheck": "bun check"` to `package.json`. Switch it to
   `bun --check` when the project has a `check` script (next section).

## Commands

```bash
bun check                              # nearest tsconfig.json; exit 1 on any error, 0 when clean
bun check src/index.ts                 # a file and everything it imports, with its tsconfig.json
bun check -p packages/server           # another project: a tsconfig.json or its directory
bun check -b                           # follow `references` like tsc -b; nothing has to be built
bun check --noUncheckedIndexedAccess   # any compiler option as a flag; overrides tsconfig.json
bun check --no-pretty                  # one line per error (tsc --pretty false); default when piped
bun check --threads 4 --timing         # cap threads on shared runners; print load/check times
```

| Instead of | Run |
|---|---|
| `tsc --noEmit` | `bun check` |
| `tsc --noEmit -p packages/server` | `bun check -p packages/server` |
| `tsc -b` | `bun check -b` |
| `tsc --noEmit && bun src/index.ts` | `bun --check src/index.ts` |
| `tsc --noEmit && bun test` | `bun test --check` |
| `tsc --noEmit && bun build ./app.ts` | `bun build --check ./app.ts` |

The three `--check` rows check what runs and everything it imports, not every file
`tsconfig.json` includes as `tsc --noEmit` did. Keep a whole-project `bun check` (a `typecheck`
script, a CI step) for files no entry point reaches.

A run that finds nothing to check exits **1** (`Nothing to check: no TypeScript files`), so a CI
step pointed at the wrong directory fails instead of passing.

## Check, Then Run, Test, or Build

`--check` type checks first. On any error Bun prints it, exits 1, and does nothing else.

```bash
bun --check src/index.ts                       # same as bun run --check src/index.ts
bun run --check dev                            # a package.json script: checks the whole project
bun test --check                               # test files + imports; no test runs on an error
bun build --check ./src/index.ts --outdir out  # nothing is written on an error
bun --watch --check src/index.ts               # re-check before every restart
```

```typescript
const result = await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './out',
  check: true,          // v1.4.3+
  throw: false,
})
// On a type error: result.success === false, no outputs, and each log is a
// BuildMessage whose message starts with the code: "TS2322: Type 'string' is ..."
```

`--check` follows static imports from what runs. It does **not** cover a `new Worker("./w.ts")`,
a computed `import(name)`, `require()` in a TypeScript file, a spawned process, or `--preload`
files. A `package.json` script, a `node_modules/.bin` executable, or a shell script gets a
whole-project check instead. `bun check` covers everything `tsconfig.json` includes. There is
no `bun check --watch`; use `bun --watch --check`.

## A `check` Script Shadows the Type Checker

When `package.json` has a `check` script, `bun check` runs that script, like `bun run check`.
`bun --check` with no file then type checks the project.

```json
{ "scripts": { "check": "biome check", "typecheck": "bun --check" } }
```

```bash
bun check         # runs "biome check"
bun --check       # type checks the project
bun run typecheck # type checks the project
```

Only inside the `check` script itself is `bun check` the type checker, so `"check": "bun check"`
works on 1.4.3 -- but an older Bun, including an old `bun` in `node_modules/.bin` that scripts
find first, runs it again and again without end. Inside any other script `bun check` still runs
the `check` script: with the scripts above, `"typecheck": "bun check"` (what `bun init` writes)
would run `biome check`. In scripts, write `bun --check`. Under `--filter` / `--workspaces`,
`check` always means each package's `check` script.

## Output

| Where | Format |
|---|---|
| Terminal | Source excerpt with notes; above 50, identical errors are grouped (`--all` shows every one) |
| Piped or redirected | `src/index.ts(3,21): error TS2322: ...` on stdout, the `Found N errors ...` summary on stderr |
| `AGENT=1`, `CLAUDECODE=1`, or `REPL_ID=1` | Tagged blocks (below); `--no-pretty` switches back to one line per error |
| GitHub Actions | One line per error plus an `::error file=...` annotation per error; no problem matcher needed |

```text
<error file="src/index.ts" line="3" column="21" code="TS2322">
Type 'string' is not assignable to type 'number'.
<source>
3 | console.log(greet({ id: "1", name: "Ada" }));
                        ^^
</source>
<related file="src/user.ts" line="2" column="3">The expected type comes from property 'id' which is declared here on type 'User'</related>
</error>
```

A clean run prints `✓ No type errors in N files`. A note names the fix when a cause is
environmental: `Run: bun add -d @types/bun` when Bun's types are missing (with `bun-types`,
add `"types": ["bun-types"]` instead), or `Stopped before type checking N files` when a syntax
error or an invalid option ended the run before any type was checked.

## TypeScript 7 Defaults

`bun check` behaves like TypeScript 7 whichever `typescript` version is installed. A project
written for TypeScript 5 hits three changed defaults:

| Default | Symptom | Fix |
|---|---|---|
| `strict` is on | `TS7006: Parameter 'x' implicitly has an 'any' type` | Fix, or set `"strict": false` |
| `types` is empty | `Cannot find name 'process'`, `'describe'`, `'Bun'` | List every types package: `"types": ["bun-types"]` (it brings Node's types) |
| Side-effect imports are checked | `TS2882` on `import "./index.css"` | Declare the file type: `declare module "*.css" {}` |

A removed option stops the check before any file is checked, exactly as `tsc` 7 does:

| In `tsconfig.json` | Replace with |
|---|---|
| `"moduleResolution": "node"` / `"node10"` / `"classic"` | `"bundler"`; `"nodenext"` for code Node runs unbundled |
| `"baseUrl": "."` | Remove it, and start each `paths` entry with `./` |
| `"target": "es5"` | `"es2015"` or later |
| `"module": "amd"` / `"umd"` / `"system"` | `"preserve"`, `"nodenext"`, or `"commonjs"` |
| `"downlevelIteration"`, `"outFile"` | Remove |
| `"esModuleInterop": false`, `"alwaysStrict": false` | Remove -- both are always on |

## Monorepos

- `bun check` checks the project of the nearest `tsconfig.json`, like `tsc`. Referenced
  projects are read from source, so nothing has to be built first.
- `-b` checks every project in `references`, in build order, each with its own options.
  Compiler flags apply to all of them (`tsc -b --strict` is an error; `bun check -b --strict`
  is not).
- A root `tsconfig.json` with only `references` (the `create vite` layout) is followed rather
  than reported as empty.
- No `tsconfig.json` at or above the directory, but some below: Bun lists them and stops.
  `bun check .` checks every TypeScript file below, each with its nearest `tsconfig.json`.
- `bun --filter '*' typecheck` runs each workspace's own script.

## CI

```yaml
- uses: oven-sh/setup-bun@v2
- run: bun install --frozen-lockfile
- run: bun check          # or: bun test --check, to gate the tests on types in one step
```

`bun check` writes no `.tsbuildinfo`, so there is no cache to save or restore -- every run
checks the whole project. For a pre-commit hook, `bun check <staged files>` checks those files
and their imports with the project's `tsconfig.json`, but a change can break a file that
imports it, so keep the whole-project run in CI.

## Differences From tsc

- **Checks only.** No JavaScript, `.d.ts`, source map, or `.tsbuildinfo`, even with
  `declaration` or `incremental` on.
- **File arguments keep `tsconfig.json`.** `tsc src/index.ts` reports `TS5112` next to a
  `tsconfig.json`; `bun check src/index.ts` checks the file with it.
- **The `lib.*.d.ts` files are built in**, with paths such as `bundled:///libs/lib.dom.d.ts`.
- **Language service `plugins` are ignored, and Yarn Plug'n'Play is unsupported** -- set
  `nodeLinker: node-modules` in `.yarnrc.yml`.
- **A mismatch with `tsc` 7 is a Bun bug.** Compare
  `bunx -p typescript@7.0.2 tsc --noEmit --pretty false` with `bun check --no-pretty` and report
  the difference upstream.
