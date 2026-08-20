# Running and Execution Reference

> `bun --help` and `bun run --help` are authoritative for flags. Concepts:
> `node_modules/bun-types/docs/runtime/` (`environment-variables.mdx`, `watch-mode.mdx`,
> `debugger.mdx`, `repl.mdx`, `bunfig.mdx`) and `docs/pm/bunx.mdx`.

## bun repl

Native REPL, built into the binary since v1.3.10 (it used to download an npm package on first
run). Syntax highlighting, persistent history at `~/.bun_repl_history`, tab completion,
multi-line continuation, `.help`/`.load`/`.save`/`.editor`, top-level `await`, and the `_` /
`_error` variables. Bare object literals need no wrapping parens.

```bash
bun repl
bun repl -e 'console.log(1 + 1)'      # evaluate
bun repl -p '{ a: 1, b: 2 }'          # evaluate and print
bun repl -p 'await fetch("https://bun.sh").then(r => r.status)'
```

`bun --interactive` starts a Node.js-compatible REPL instead.

## bun run

Run package.json scripts or files.

```bash
bun run [flags] <script|file> [args...]
```

| Flag | Description |
|---|---|
| `--watch` | Re-run on file changes |
| `--hot` | Hot reload (preserves application state) |
| `--smol` | Reduce memory usage at the cost of throughput |
| `--silent` | Don't echo script name to stderr |
| `--env-file path` | Load env file(s); repeatable, left takes priority |
| `--shell bun\|system` | Which shell to use for scripts (macOS/Linux default: system; Windows: bun) |
| `--bun` | Force Bun runtime instead of Node.js |
| `--cwd path` | Set working directory |
| `--filter pattern` | Run in specific workspace(s) |
| `--if-present` | Don't error if script is missing |
| `--preconnect url` | Pre-connect to server URLs for faster startup (repeatable) |
| `--no-orphans` | Exit when the parent process dies, then SIGKILL every descendant (Linux/macOS, v1.3.14+) |

### Script Resolution Order

When you run `bun <name>`:
1. package.json `"scripts"` field
2. `node_modules/.bin/` binaries
3. File on disk (with TypeScript/JavaScript extensions)
4. `bunfig.toml` aliases

### Watch vs Hot

- `--watch` — Kills and restarts the process on file changes. Clean state.
- `--hot` — Reloads modules in-place without restarting. Preserves global state, open connections, etc. Uses `module.hot` API for cleanup handlers.

### Pre/Post Scripts

Bun runs `pre<script>` and `post<script>` if they exist:
```json
{
  "scripts": {
    "predev": "echo 'before dev'",
    "dev": "bun run server.ts",
    "postdev": "echo 'after dev'"
  }
}
```

## Direct File Execution

```bash
bun file.ts             # Run TypeScript
bun file.js             # Run JavaScript
bun file.jsx            # Run JSX
bun file.tsx            # Run TSX
bun file.mjs            # Run ESM JavaScript
bun file.cjs            # Run CommonJS JavaScript
bun file.md             # Render Markdown in the terminal (v1.3.12+)
```

Running a `.md` file renders it as ANSI-colored terminal output (via `Bun.markdown.ansi()`) with no JavaScript VM startup overhead -- a fast built-in Markdown viewer.

### Flags for Direct Execution

All flags available when running files directly:

| Flag | Description |
|---|---|
| `--watch` | Re-run on changes |
| `--hot` | Hot reload |
| `--smol` | Reduce memory usage |
| `--env-file path` | Load env file |
| `--preload module` | Preload modules before execution |
| `--inspect` | Enable debugger |
| `--inspect-wait` | Wait for debugger before execution |
| `--inspect-brk` | Break on first line |
| `--cpu-prof` | CPU profiling |
| `--cpu-prof-md` | CPU profiling in Markdown format (v1.3.7+) |
| `--heap-prof` | Heap profiling |
| `--heap-prof-md` | Heap profiling in Markdown format (v1.3.7+) |
| `--conditions name` | Custom export conditions |
| `--tsconfig-override path` | Use specific tsconfig.json |
| `--define K=V` | Compile-time defines |
| `--loader .ext=type` | Custom file loaders |
| `--main-fields field1,field2` | Override package.json main fields |
| `--extension-order .ts,.js` | Override module resolution order |
| `--prefer-offline` | Prefer cached packages |
| `--prefer-latest` | Prefer latest packages |
| `--no-install` | Disable auto-install |
| `-e, --eval code` | Evaluate string as script |
| `--print code` | Evaluate and print result |
| `--experimental-http2-fetch` | Offer HTTP/2 (h2) in `fetch()` TLS ALPN (v1.3.14+) |
| `--experimental-http3-fetch` | Honor `Alt-Svc: h3` and upgrade `fetch()` to HTTP/3 (v1.3.14+) |
| `--use-system-ca` | Trust the system's certificate authorities (v1.3.14+) |

## bunx

Execute package binaries, auto-installing if needed.

```bash
bunx [flags] <package[@version]> [args...]
```

| Flag | Description |
|---|---|
| `--bun` | Force Bun runtime |
| `--cwd path` | Set working directory |
| `-p, --packages pkg` | Additional packages to install |

### Resolution Order

1. Local `node_modules/.bin/`
2. Global cache
3. Auto-install from registry

### Examples

```bash
bunx prettier --write .
bunx --bun next dev
bunx create-react-app my-app
bunx tsc --noEmit
bunx -p typescript tsc --noEmit     # Install typescript, run tsc
bunx cowsay@1.0.0 "hello"          # Specific version
bunx claude                        # Built-in alias for @anthropic-ai/claude-code (v1.3.13+)
```

## bun exec

Run a shell command with Bun's shell (cross-platform).

```bash
bun exec "shell command here"
```

Uses Bun's built-in shell, which works consistently across macOS, Linux, and Windows.

## Zero-Config Frontend Development

Run HTML files directly with Bun for instant frontend development -- no bundler config, no dev server setup.

```bash
bun ./index.html                   # Start dev server for HTML file
bun --hot ./index.html             # With hot module replacement
```

Bun automatically:
- Serves the HTML file with a local dev server
- Transpiles linked JavaScript, TypeScript, JSX, and TSX files
- Processes CSS imports
- Enables Hot Module Replacement (HMR) and React Fast Refresh
- Resolves `node_modules` imports in `<script>` tags

### Example HTML

```html
<!DOCTYPE html>
<html>
<head>
  <link rel="stylesheet" href="./styles.css">
</head>
<body>
  <div id="root"></div>
  <script type="module" src="./app.tsx"></script>
</body>
</html>
```

Running `bun ./index.html` serves this with all linked assets automatically bundled and served. TypeScript and JSX are transpiled on the fly. No `vite.config.ts`, `webpack.config.js`, or any configuration needed.

## Parallel and Sequential Script Execution

Run multiple scripts concurrently or in strict order:

```bash
bun run --parallel build lint typecheck    # Run all three concurrently
bun run --parallel 'build:*'               # Glob-match script names
bun run --parallel --filter '*' build      # Fan out across every workspace
bun run --parallel --no-exit-on-error --filter '*' test   # Keep going past failures
bun run --sequential clean build deploy    # One at a time, same prefixed output
```

Replaces `concurrently` and `npm-run-all`. Each output line is prefixed with the script name,
or `package:script` under `--filter`, and `pre*`/`post*` hooks stay grouped with their main
script so dependency order is preserved.

The flag may also precede `run` (`bun --parallel run build lint`) -- both forms work, but
`bun run --parallel` is the documented one.

## Auto-Install

When enabled, Bun automatically installs missing imports at runtime:

```toml
# bunfig.toml
[install]
auto = true
```

Or per-run: `bun --install=fallback file.ts`

## Environment Loading

Bun auto-loads `.env` files in this order (first match wins per variable):
1. `.env.local` (not loaded when `NODE_ENV=test`)
2. `.env.{NODE_ENV}` (e.g., `.env.production`)
3. `.env`

Override with `--env-file`:
```bash
bun --env-file .env.staging file.ts
```

Multiple files (left takes priority):
```bash
bun --env-file .env.local --env-file .env file.ts
```

Disable automatic loading entirely (v1.3.3+) -- useful in production and CI, where variables
come from the environment and a stray `.env` should be ignored. Explicit `--env-file` arguments
are still honored.

```bash
bun --no-env-file server.ts
```

```toml
# bunfig.toml
env = false
```

**Changed in 1.4 -- Bun as `node` skips `.env`.** Under `bun --bun`, `bunx --bun`, or a `node`
symlink to Bun, `.env`, `.env.local`, and `.env.{development,production,test}` are **not**
loaded, matching Node.js. `bun file.js` still loads them. A `package.json` script calling
`node` under `bun --bun run` now sees those variables as `undefined`:

```json
{
  "scripts": {
    "check": "node --env-file=.env ./check.js"
  }
}
```
