# Running and Execution Reference

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
```

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
| `--heap-prof` | Heap profiling |
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
```

## bun exec

Run a shell command with Bun's shell (cross-platform).

```bash
bun exec "shell command here"
```

Uses Bun's built-in shell, which works consistently across macOS, Linux, and Windows.

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
