# Bundling and Compilation Reference

> `bun build --help` is authoritative for flags. Concepts:
> `node_modules/bun-types/docs/bundler/**/*.mdx` (`index.mdx`, `executables.mdx`,
> `loaders.mdx`, `plugins.mdx`, `bytecode.mdx`, `macros.mdx`, `standalone-html.mdx`).
> 1.4 changes: `migration-1.4.md`.

## bun build

Bundle TypeScript/JavaScript for distribution.

```bash
bun build [flags] <entrypoint(s)>
```

### Output Flags

| Flag | Description |
|---|---|
| `--outdir path` | Output directory |
| `--outfile path` | Output to single file (mutually exclusive with --outdir) |
| `--splitting` | Enable code splitting (ESM format only) |
| `--min-chunk-size N` | With `--splitting`, fold side-effect-free chunks smaller than N source bytes into their loader; `16384` suits browser builds (v1.4.1+) |
| `--no-module-preload` | With `--splitting --target browser`, skip the default `<link rel="modulepreload">` links (v1.4.1+) |
| `--no-split-require` | With `--splitting --target bun`, inline a `require()`'d ES module instead of emitting a chunk (v1.4.1+) |
| `--no-deprecated-namespace-object-setters` | Getter-only `import * as ns` objects; the future default (v1.4.1+) |
| `--sourcemap mode` | `external`, `inline`, `linked`, `none` (default: none) |
| `--minify` | Minify all (shorthand for below three flags) |
| `--minify-syntax` | Minify syntax only |
| `--minify-whitespace` | Minify whitespace only |
| `--minify-identifiers` | Minify identifiers only |
| `--root path` | Root directory for computing entry point paths in output |

### Code Splitting (v1.4.1 behavior)

With `--splitting`, code shared between an entry and its lazy routes stays in the entry's chunk,
browser builds get `<link rel="modulepreload">` for every chunk an entry or `import()` loads,
`require()` of an ES module is its own chunk under `--target bun`, and `export * as` /
`const { x } = await import()` tree-shake unused exports. Details and the 1.4.0 diff:
`migration-1.4.md`.

### Target and Format

| Flag | Description |
|---|---|
| `--target` | `browser` (default), `bun`, `node` |
| `--format` | `esm` (default), `cjs`, `iife` |

**Targets:**
- `browser` — For web browsers. Strips `node:` imports, polyfills APIs.
- `bun` — For Bun runtime. Marks Bun-native modules as external.
- `node` — For Node.js. Marks Node built-ins as external.

**Formats:**
- `esm` — ES Modules (import/export)
- `cjs` — CommonJS (require/module.exports)
- `iife` — Immediately Invoked Function Expression (for `<script>` tags)

### Module Resolution

| Flag | Description |
|---|---|
| `--external pkg` | Exclude from bundle (repeatable) |
| `--packages external` | Treat all packages as external |
| `--packages bundle` | Bundle all packages (default for browser) |
| `--conditions name` | Custom export conditions (repeatable) |
| `--main-fields f1,f2` | Override package.json main field resolution |
| `--extension-order .ts,.js` | Override module resolution extension order |
| `--resolve=bun` | Use Bun's module resolution |
| `--resolve=node` | Use Node.js module resolution |

### Naming and Paths

| Flag | Description |
|---|---|
| `--entry-naming pattern` | Pattern for entry point output names |
| `--chunk-naming pattern` | Pattern for chunk output names |
| `--asset-naming pattern` | Pattern for asset output names |
| `--public-path prefix` | URL prefix for assets (e.g., `/cdn/`) |

Naming patterns support: `[name]`, `[ext]`, `[hash]`, `[dir]`.

Default: `[dir]/[name].[ext]`

### Defines and Loaders

| Flag | Description |
|---|---|
| `--define K=V` | Compile-time constant replacement (repeatable) |
| `--loader .ext=type` | Map file extension to loader (repeatable) |
| `--jsx-runtime automatic\|classic` | JSX transform mode |
| `--jsx-factory name` | JSX factory function (classic mode) |
| `--jsx-fragment name` | JSX fragment (classic mode) |
| `--jsx-import-source pkg` | JSX import source (automatic mode) |
| `--tsconfig-override path` | Use specific tsconfig.json |

**Loaders:** `js`, `jsx`, `ts`, `tsx`, `json`, `toml`, `css`, `text`, `file`, `base64`, `dataurl`, `binary`, `napi`, `wasm`

### Banner and Footer

| Flag | Description |
|---|---|
| `--banner:js "code"` | Prepend to JS output |
| `--footer:js "code"` | Append to JS output |
| `--banner:css "code"` | Prepend to CSS output |
| `--footer:css "code"` | Append to CSS output |

### Other Flags

| Flag | Description |
|---|---|
| `--drop name` | Remove function calls (e.g., `--drop console` removes console.*) |
| `--ignore-dce-annotations` | Ignore `/* @__PURE__ */` and `sideEffects` |
| `--tree-shaking` | Enable/disable tree shaking (default: true for production) |
| `--manifest` | Generate build manifest file |
| `--metafile [path]` | Bundle metadata in esbuild's format -- works with esbuild's analyzer (v1.3.6+) |
| `--metafile-md [path]` | Module graph as a Markdown report: summary, largest inputs, per-entry breakdown, dependency chains (v1.3.8+) |
| `--server-components` | Enable React Server Components support |
| `--css-chunking` | Enable CSS code splitting |
| `--emit-dce-annotations` | Emit dead code elimination annotations |
| `--react-compiler` | Run React's auto-memoization compiler inside Bun's parser -- no Babel or SWC (v1.4+) |
| `--react-fast-refresh` | React Fast Refresh transform |
| `--feature NAME` | Compile-time flag for `feature()` from `bun:bundle`; the dead branch is removed |
| `--allow-unresolved glob` | Permit unresolved dynamic `import()`/`require()` specifiers (default `*`) |
| `--reject-unresolved` | Fail the build on any unresolvable dynamic specifier |
| `--banner`, `--footer` | Prepend/append text (e.g. `"use client"`) |
| `--keep-names` | Preserve function and class names when minifying |
| `--packages external\|bundle` | Keep dependencies external or bundle them |

**Changed in 1.4:** `--metafile` now sets a bundled import's `path` to the imported file's
`inputs` key, so `metafile.inputs[path]` matches (it was the raw specifier or an absolute path).
`--minify` no longer emits a bare `$` identifier, which shadowed jQuery in classic scripts. A
bundled `import * as ns` namespace enumerates its exports in sorted order -- update snapshots
that pinned the old order.

### Bun.build() Options Without a CLI Flag

```typescript
await Bun.build({
  entrypoints: ['./src/index.tsx'],
  reactCompiler: true,                          // same as --react-compiler (v1.4+)
  optimizeImports: ['antd', '@mui/material'],   // prune barrel files (v1.3.10+)
  metafile: true,                               // result.metafile, esbuild format
  files: {                                      // bundle from memory (v1.3.6+)
    '/app/index.ts': `import { greet } from "./greet.ts"; greet("World")`,
    '/app/greet.ts': `export function greet(n: string) { return "Hello, " + n }`,
  },
})
```

`optimizeImports` skips the hundreds of files behind names you did not import from a barrel
package. Packages declaring `"sideEffects": false` get this automatically; everything else
needs the opt-in. `files` maps paths to strings, `Blob`s, or `TypedArray`s, and virtual paths
take precedence over real ones -- handy for codegen and for stubbing a module in tests.

### Compile-Time Feature Flags

```typescript
import { feature } from 'bun:bundle'

if (feature('SUPER_SECRET')) { /* removed unless --feature=SUPER_SECRET */ }
```

Works in `bun build`, `bun run`, and `bun test`. Set via `--feature=FLAG` or `features: [...]`
in `Bun.build()`.

## Standalone Executables (--compile)

Create self-contained executables that include Bun runtime.

```bash
bun build --compile [flags] <entrypoint>
```

### Compile-Specific Flags

| Flag | Description |
|---|---|
| `--compile` | Create standalone executable (implies `--production`) |
| `--target platform` | Cross-compile target (see below) |
| `--outfile name` | Output executable name |
| `--minify` | Minify bundled code |
| `--asset path` | Embed a file or directory, preserving relative paths (v1.4+) |
| `--bytecode` | Bytecode cache; supports ES modules with `--format=esm` as of v1.3.9; about 3x source size as of v1.4.1 |
| `--bytecode-depth N` | Compile only the top N nesting levels to bytecode ahead of time; `0` is top-level only, deeper functions compile on first call. `bytecodeDepth` in `Bun.build()` with `bytecode: true` and `target: 'bun'` (v1.4.1+) |
| `--compile-exec-argv args` | Prepend arguments to the executable's `execArgv` |
| `--compile-executable-path path` | Use a local Bun binary instead of downloading one when cross-compiling (v1.3.6+) |
| `--compile-autoload-tsconfig` | Re-enable runtime `tsconfig.json` loading (**off by default since v1.3.4**) |
| `--compile-autoload-package-json` | Re-enable runtime `package.json` loading (**off by default since v1.3.4**) |
| `--no-compile-autoload-dotenv` | Disable `.env` autoloading (on by default) |
| `--no-compile-autoload-bunfig` | Disable `bunfig.toml` autoloading (on by default) |
| `--windows-icon`, `--windows-title`, `--windows-publisher`, `--windows-version`, `--windows-description`, `--windows-copyright`, `--windows-hide-console` | Windows executable metadata |

**Changed in v1.3.4:** compiled binaries no longer auto-load `tsconfig.json` or `package.json`
from the runtime working directory, so a binary can no longer pick up unrelated config from
wherever it happens to run. `.env` and `bunfig.toml` still auto-load.

### Embedding Assets (v1.4+)

`--asset` embeds a file or a whole directory into the executable with original filenames, so
`path.join(import.meta.dir, ...)` resolves the same as on disk.

```bash
bun build ./build/index.js --compile \
  --asset ./build/client --asset ./build/prerendered \
  --outfile server
```

`node:fs` treats `/$bunfs/` as a real directory tree -- `existsSync`, `statSync`, `lstatSync`,
`accessSync`, `readdirSync`, and `fs.promises.readdir` (including `withFileTypes` and
`recursive`) all work on embedded paths. Static file servers that enumerate a directory at
startup run unmodified inside the binary.

Check whether you are inside one with `Bun.isStandaloneExecutable` (v1.4+) -- unlike
`Bun.embeddedFiles.length > 0`, reading it allocates nothing.

### Cross-Compilation Targets

```bash
bun build --compile --target bun-linux-x64 ./app.ts
bun build --compile --target bun-linux-arm64 ./app.ts
bun build --compile --target bun-darwin-x64 ./app.ts
bun build --compile --target bun-darwin-arm64 ./app.ts
bun build --compile --target bun-windows-x64 ./app.ts
```

On Linux (v1.3.12+), the runtime is embedded via a dedicated `.bun` ELF section instead of being read from `/proc/self/exe`, which allows execute-only (non-readable) standalone binaries.

Cross-compiling with `--bytecode` works for every target as of v1.4.1, `bun-windows-x64` from
macOS or Linux included. The bytecode cache format is identical on all platforms, so builds are
byte-identical.

Bun's runtime also ships native first-party builds for FreeBSD and Android (v1.3.14+), alongside Linux, macOS, and Windows -- these are runtime platforms, not `--compile` targets.

### Browser Target (v1.3.10+)

Compile to a self-contained HTML file that runs in the browser:

```bash
bun build --compile --target=browser ./app.tsx --outfile ./dist/app.html
```

Produces a single HTML file with all JS/CSS inlined -- useful for distributing single-file web apps. As of v1.3.13, file-loader assets (images, fonts) are also inlined as data URIs, so the output is truly self-contained.

### Embedding Assets

Files can be embedded in the executable:
```typescript
// In your code
const file = Bun.file(import.meta.dir + '/data.json')
const data = await file.json()
```

Use `--public-path` to include asset files alongside the executable.

## Programmatic API

```typescript
const result = await Bun.build({
  entrypoints: ['./src/index.ts'],
  outdir: './dist',
  target: 'browser',           // 'browser' | 'bun' | 'node'
  format: 'esm',               // 'esm' | 'cjs' | 'iife'
  splitting: true,
  minChunkSize: 16 * 1024,     // fold small side-effect-free chunks into their loader (v1.4.1+, default 0)
  modulePreload: true,         // default; false skips <link rel="modulepreload"> (v1.4.1+)
  splitRequire: true,          // default with target 'bun'; false inlines require()'d ESM (v1.4.1+)
  deprecatedNamespaceObjectSetters: false,  // getter-only namespace objects, the future default (v1.4.1+)
  sourcemap: 'external',       // 'external' | 'inline' | 'linked' | 'none'
  minify: true,                // or { syntax: true, whitespace: true, identifiers: true }
  external: ['react'],
  define: { 'process.env.NODE_ENV': '"production"' },
  naming: {
    entry: '[dir]/[name].[ext]',
    chunk: '[name]-[hash].[ext]',
    asset: '[name]-[hash].[ext]',
  },
  publicPath: '/cdn/',
  loader: { '.svg': 'text' },
  metafile: true,              // Generate bundle analysis metadata
  plugins: [myPlugin],
  root: './src',
  banner: '/* banner */',
  footer: '/* footer */',
  drop: ['console', 'debugger'],
  treeshaking: true,
})

if (!result.success) {
  for (const log of result.logs) {
    console.error(log)
  }
}

for (const output of result.outputs) {
  console.log(output.path)   // Output file path
  console.log(output.kind)   // 'entry-point' | 'chunk' | 'asset'
  console.log(output.size)   // Size in bytes
}
```

## Build Plugins

```typescript
import type { BunPlugin } from 'bun'

const myPlugin: BunPlugin = {
  name: 'my-plugin',
  setup(build) {
    // Filter and transform modules
    build.onLoad({ filter: /\.yaml$/ }, async (args) => {
      const text = await Bun.file(args.path).text()
      const data = parseYaml(text)
      return {
        contents: `export default ${JSON.stringify(data)}`,
        loader: 'js',
      }
    })

    // Resolve custom imports
    build.onResolve({ filter: /^virtual:/ }, (args) => {
      return { path: args.path, namespace: 'virtual' }
    })
  },
}
```
