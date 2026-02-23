# Bundling and Compilation Reference

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
| `--sourcemap mode` | `external`, `inline`, `linked`, `none` (default: none) |
| `--minify` | Minify all (shorthand for below three flags) |
| `--minify-syntax` | Minify syntax only |
| `--minify-whitespace` | Minify whitespace only |
| `--minify-identifiers` | Minify identifiers only |
| `--root path` | Root directory for computing entry point paths in output |

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
| `--server-components` | Enable React Server Components support |
| `--css-chunking` | Enable CSS code splitting |
| `--emit-dce-annotations` | Emit dead code elimination annotations |

## Standalone Executables (--compile)

Create self-contained executables that include Bun runtime.

```bash
bun build --compile [flags] <entrypoint>
```

### Compile-Specific Flags

| Flag | Description |
|---|---|
| `--compile` | Create standalone executable |
| `--target platform` | Cross-compile target (see below) |
| `--outfile name` | Output executable name |
| `--minify` | Minify bundled code |

### Cross-Compilation Targets

```bash
bun build --compile --target bun-linux-x64 ./app.ts
bun build --compile --target bun-linux-arm64 ./app.ts
bun build --compile --target bun-darwin-x64 ./app.ts
bun build --compile --target bun-darwin-arm64 ./app.ts
bun build --compile --target bun-windows-x64 ./app.ts
```

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
