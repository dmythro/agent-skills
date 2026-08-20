---
name: bun-api
description: >-
  Bun runtime API reference for TypeScript scripts. Covers Bun.file(), Bun.write(),
  Bun.$() shell, Bun.spawn(), Bun.Glob, Bun.env, bun:sqlite, Bun.sql() for PostgreSQL/MySQL
  via DATABASE_URL, Bun.s3 for S3-compatible storage, Bun.redis for Redis/Valkey,
  Bun.Archive for tarballs, Bun.Image image processing, Bun.WebView headless browser
  automation, Bun.cron scheduling, Bun.secrets OS credential storage, Bun.Terminal PTYs,
  JSONC/JSON5/JSONL/XML/TOML/YAML/markdown parsers,
  Bun.hash, Bun.password, compression, and scripting utilities. Use when writing scripts,
  automating tasks, querying databases, working with S3 storage, Redis caching, processing
  images, automating a headless browser, parsing markdown/JSON variants, or doing file
  processing in a Bun project. Signals: bun.lock, bunfig.toml,
  DATABASE_URL, REDIS_URL, AWS_ACCESS_KEY_ID, Bun.$ usage
  Not for bun CLI commands (bun-cli skill), non-Bun runtimes, or ORM CLI tooling
---

# Bun Runtime API

Bun runs TypeScript natively — no `tsc` compilation, no `ts-node`, no build step. Run any `.ts` file directly with `bun file.ts`. Use Bun's native APIs instead of Node.js equivalents — they're faster, more ergonomic, and require no additional dependencies.

**Critical**: In a Bun project (has `bun.lock`, `bun.lockb`, `bunfig.toml`, or `@types/bun` in devDependencies), always use Bun to run scripts (`bun file.ts`, not `node file.ts`) and prefer Bun-native APIs over Node.js equivalents. Mixing runtimes causes subtle bugs and unnecessary retries.

**Verified against Bun v1.4.0** (2026-08-20). Features are tagged with the version that
introduced them (`v1.4+`). Where 1.4 changed existing behavior, both behaviors are stated
so this skill stays correct on 1.3.x projects -- check `bun --version` before relying on a
version-tagged item.

## Read Bun's Own Docs First

Bun ships its complete documentation inside `bun-types`, version-matched to the runtime.
In any project with `bun-types` or `@types/bun` installed:

```text
node_modules/bun-types/docs/**/*.mdx   # full docs, plus ~180 task-shaped guides/
node_modules/bun-types/*.d.ts          # richest API surface (bun.d.ts, serve.d.ts, sql.d.ts)
node_modules/bun-types/CLAUDE.md       # Bun's own agent rules
```

**Consult them before writing non-trivial Bun code.** This skill covers which API to reach
for; the shipped docs cover exact signatures and options.

1. **Check the version.** Compare `bun --version` against `node_modules/bun-types/package.json`.
   `bun init` installs `@types/bun@latest`, which lags behind the runtime -- correct it with
   `bun add -d bun-types@<runtime-version>`.
2. **Open by explicit path.** With Bun's global virtual store enabled, `node_modules/bun-types`
   is a symlink: `find node_modules -name '*.mdx'` and `rg <pattern> node_modules` return
   nothing, while `find node_modules/bun-types/docs -name '*.mdx'` works.
3. **Never edit files under `node_modules/`.** Under the global store, every project on the
   machine shares the same inode -- a write there hits all of them. Use `bun patch`.
4. **Not installed?** The same path works online: `docs/runtime/sql.mdx` is
   `https://bun.com/docs/runtime/sql`.

| Task | Doc path (under `node_modules/bun-types/docs/`) |
|---|---|
| HTTP server, routes, WebSockets | `runtime/http/server.mdx`, `runtime/http/routing.mdx`, `runtime/http/websockets.mdx` |
| `fetch`, TCP, UDP, DNS | `runtime/networking/fetch.mdx`, `networking/tcp.mdx`, `networking/udp.mdx`, `networking/dns.mdx` |
| File I/O, streams, binary data | `runtime/file-io.mdx`, `runtime/streams.mdx`, `runtime/binary-data.mdx` |
| Shell, subprocesses, PTY | `runtime/shell.mdx`, `runtime/child-process.mdx` |
| SQL, SQLite, Redis, S3 | `runtime/sql.mdx`, `runtime/sqlite.mdx`, `runtime/redis.mdx`, `runtime/s3.mdx` |
| Parsers | `runtime/json5.mdx`, `jsonl.mdx`, `xml.mdx`, `toml.mdx`, `yaml.mdx`, `markdown.mdx`, `file-types.mdx` |
| Images, WebView, cron, secrets, archives | `runtime/image.mdx`, `webview.mdx`, `cron.mdx`, `secrets.mdx`, `archive.mdx` |
| Hashing, utils, semver, glob, cookies, CSRF | `runtime/hashing.mdx`, `utils.mdx`, `semver.mdx`, `glob.mdx`, `cookies.mdx`, `csrf.mdx` |
| Node.js compatibility | `runtime/nodejs-compat.mdx` |

## When to Use

- Scripts for generating files, parsing data, running migrations
- File processing and transformation pipelines
- Shell scripting and automation
- Database operations with SQLite (`bun:sqlite`)
- **Database queries via connection URL** -- project has `DATABASE_URL` in `.env` or environment (PostgreSQL, MySQL, SQLite via `Bun.sql()`)
- **S3 storage operations** -- project has `AWS_ACCESS_KEY_ID` or uses S3-compatible storage (`Bun.s3`)
- **Redis/Valkey caching and pub/sub** -- project has `REDIS_URL` or `VALKEY_URL` (`Bun.redis`)
- Any scripting task in a Bun project

## HTTP Server (Bun.serve)

Built-in HTTP server — replaces Express, Fastify, or `http.createServer`.

Prefer `routes` over hand-rolled `URL` parsing -- it gives you params, per-method handlers,
and zero-allocation static responses. `fetch` is the fallback for unmatched requests.

```typescript
const server = Bun.serve({
  port: 3000,

  routes: {
    '/health': new Response('OK'),                    // static, zero-allocation
    '/api/users/:id': req => Response.json({ id: req.params.id }),
    '/api/posts': {                                   // per-method handlers
      GET: () => Response.json(listPosts()),
      POST: async req => Response.json(await req.json()),
    },
    '/static/*': { dir: './public' },                 // serve a directory (v1.4+)
    '/*': Response.json({ error: 'not found' }, { status: 404 }),
  },

  fetch(req: Request): Response | Promise<Response> {  // unmatched requests
    return new Response('Not Found', { status: 404 })
  },

  error(error: Error): Response {
    return new Response(`Error: ${error.message}`, { status: 500 })
  },
})

console.log(`Listening on ${server.url}`)
```

Route precedence: exact > `:param` > `*` > global `/*`. Handlers receive a `BunRequest`
(a `Request` plus `params` and `cookies`).

Key methods: `server.stop()`, `server.reload()` (hot-swap handler), `server.requestIP(req)`, `server.upgrade(req)` (WebSocket).

> **Reference**: See `references/http-server.md` for TLS, WebSocket upgrade, streaming
> responses, static file serving, and 1.4 behavior changes. Full API in
> `node_modules/bun-types/docs/runtime/http/server.mdx` and `http/routing.mdx`.

## TCP / UDP Sockets

Raw sockets for non-HTTP protocols -- `Bun.listen()` / `Bun.connect()` for TCP, `Bun.udpSocket()` for UDP, plus the built-in `WebSocket` client and `fetch()`.

```typescript
const server = Bun.listen({
  hostname: '127.0.0.1',
  port: 8080,
  socket: {
    open(socket) { socket.write('welcome\n') },
    data(socket, data) { /* Buffer */ },
  },
})
```

> **Reference**: See `references/networking.md` for TCP/UDP handlers, Unix sockets, the WebSocket client (`ws+unix://`), and `fetch()` transport options (HTTP/2, HTTP/3, proxies, system CA).

## File I/O

### Reading Files

```typescript
// Create a BunFile reference (lazy, no read yet)
const file = Bun.file('path/to/file.txt')

// Read contents
const text = await file.text()           // string
const json = await file.json()           // parsed JSON
const bytes = await file.arrayBuffer()   // ArrayBuffer
const stream = file.stream()             // ReadableStream
const blob = await file.blob()           // Blob

// File metadata
file.size                                // Size in bytes
file.type                                // MIME type (auto-detected)
file.name                                // File path
await file.exists()                      // Boolean

// Read from URL
const remote = Bun.file('https://example.com/data.json')
```

### Writing Files

```typescript
// Write string
await Bun.write('output.txt', 'content')

// Write from BunFile (efficient copy)
await Bun.write('copy.txt', Bun.file('original.txt'))

// Write JSON
await Bun.write('data.json', JSON.stringify(data, null, 2))

// Write Uint8Array / ArrayBuffer
await Bun.write('binary.dat', new Uint8Array([1, 2, 3]))

// Write Response body
await Bun.write('page.html', await fetch('https://example.com'))

// Write to stdout
await Bun.write(Bun.stdout, 'Hello\n')
```

### Stdio

```typescript
Bun.stdin    // BunFile for stdin
Bun.stdout   // BunFile for stdout
Bun.stderr   // BunFile for stderr

// Read all of stdin
const input = await Bun.stdin.text()

// Stream stdin line by line
for await (const chunk of Bun.stdin.stream()) {
  // process chunk (Uint8Array)
}
```

### Common Patterns

```typescript
// JSON transform
const data = await Bun.file('input.json').json()
data.version = '2.0.0'
await Bun.write('output.json', JSON.stringify(data, null, 2))

// File generation from template
const template = await Bun.file('template.html').text()
const output = template.replace('{{title}}', 'My Page')
await Bun.write('index.html', output)

// Check if file exists before reading
const file = Bun.file('config.json')
if (await file.exists()) {
  const config = await file.json()
}
```

> **Reference**: See `references/file-io.md` for BunFile interface, write overloads, streaming, MIME detection, and file watching.

## Shell and Process Execution

### Bun.$ (Tagged Template Shell)

The primary way to run shell commands. Returns a promise with output.

```typescript
import { $ } from 'bun'

// Basic execution
const result = await $`ls -la`
console.log(result.text())          // stdout as string

// With interpolation (auto-escaped)
const dir = 'my folder'
await $`ls ${dir}`                   // Safe: "my folder" is properly quoted

// Output methods
const output = await $`echo hello`
output.text()                        // "hello\n"
output.json()                        // Parse stdout as JSON
output.lines()                       // string[] (splits on newlines)
output.bytes()                       // Uint8Array
output.blob()                        // Blob
output.exitCode                      // number
output.stderr                        // Buffer

// Piping
await $`cat file.txt | grep pattern | wc -l`

// Quiet mode (suppress stdout)
await $`npm install`.quiet()

// No-throw mode (don't throw on non-zero exit)
const result = await $`command-that-might-fail`.nothrow()
if (result.exitCode !== 0) {
  console.error('Failed:', result.stderr.toString())
}

// Combined
await $`risky-command`.quiet().nothrow()

// Environment variables
await $`echo $HOME`.env({ HOME: '/custom' })

// Working directory
await $`ls`.cwd('/tmp')

// Redirect to file
await $`echo hello > output.txt`
await $`cat < input.txt`

// Pipe between commands
const input = Buffer.from('hello')
await $`cat`.stdin(input)
```

### Bun.spawn (Lower-Level)

For more control over process execution.

```typescript
const proc = Bun.spawn(['command', 'arg1', 'arg2'], {
  cwd: '/path',
  env: { ...process.env, CUSTOM: 'value' },
  stdin: 'pipe',          // 'pipe' | 'inherit' | 'ignore' | BunFile | Blob | Response
  stdout: 'pipe',         // 'pipe' | 'inherit' | 'ignore' | BunFile
  stderr: 'pipe',         // 'pipe' | 'inherit' | 'ignore' | BunFile
  onExit(proc, exitCode, signalCode, error) {
    // Called when process exits
  },
})

// Write to stdin
proc.stdin.write('input data')
proc.stdin.end()

// Read stdout
const output = await new Response(proc.stdout).text()

// Wait for completion
await proc.exited                    // Promise<number> (exit code)

// Kill
proc.kill()                          // SIGTERM
proc.kill('SIGKILL')                 // Specific signal
```

### Bun.spawnSync (Synchronous)

```typescript
const result = Bun.spawnSync(['command', 'arg1'], {
  cwd: '/path',
  env: { ...process.env },
})

result.exitCode     // number
result.stdout       // Buffer
result.stderr       // Buffer
result.success      // boolean
```

> **Reference**: See `references/shell-and-process.md` for complete $ API, spawn options, IPC, and signal handling.

## Glob Pattern Matching

```typescript
const glob = new Bun.Glob('**/*.ts')

// Async iteration
for await (const path of glob.scan({ cwd: './src', onlyFiles: true })) {
  console.log(path)
}

// Sync iteration
for (const path of glob.scanSync('./src')) {
  console.log(path)
}

// Test if a path matches
glob.match('src/index.ts')       // true
glob.match('README.md')          // false

// Scan options
glob.scan({
  cwd: './src',                  // Directory to scan (default: '.')
  dot: false,                    // Include dotfiles (default: false)
  onlyFiles: true,               // Skip directories (default: true)
  absolute: false,               // Return absolute paths (default: false)
  followSymlinks: false,         // Follow symlinks (default: false)
})
```

## Environment and Arguments

```typescript
Bun.env.NODE_ENV                 // Environment variable (same as process.env)
Bun.env.DATABASE_URL             // Typed access

Bun.argv                         // string[] — [bunPath, scriptPath, ...args]
// Equivalent: process.argv

Bun.main                         // Absolute path to the entry point script

import.meta.dir                  // Directory of current file
import.meta.file                 // Filename of current file
import.meta.path                 // Full path of current file
import.meta.dirname              // Same as import.meta.dir (Node.js compat)
import.meta.filename             // Same as import.meta.path (Node.js compat)
```

## SQL Client (Bun.sql) -- PostgreSQL, MySQL, SQLite

Built-in SQL client for querying databases via connection URL. Zero dependencies, tagged template literals, automatic prepared statements, connection pooling. **Use when the project has `DATABASE_URL` in `.env` or environment.**

```typescript
import { sql, SQL } from "bun"

// Default instance -- auto-connects using DATABASE_URL from environment
const users = await sql`SELECT * FROM users WHERE active = ${true} LIMIT ${10}`

// Explicit connection
const db = new SQL("postgres://user:pass@localhost:5432/mydb")
const results = await db`SELECT * FROM users`

// MySQL
const mysql = new SQL("mysql://user:pass@localhost:3306/mydb")
```

### Insert / Update with Object Helpers

```typescript
const user = { name: "Alice", email: "alice@example.com" }

// Insert -- expands object to (column1, column2) VALUES (val1, val2)
const [newUser] = await sql`INSERT INTO users ${sql(user)} RETURNING *`

// Bulk insert
await sql`INSERT INTO users ${sql([user1, user2, user3])}`

// Update -- expands to SET column1 = val1, column2 = val2
await sql`UPDATE users SET ${sql(updates)} WHERE id = ${userId}`
```

### Transactions

```typescript
await sql.begin(async (tx) => {
  const [user] = await tx`INSERT INTO users (name) VALUES (${"Alice"}) RETURNING *`
  await tx`INSERT INTO audit_log (action, user_id) VALUES ('created', ${user.id})`
})
// Auto-committed on success, rolled back on error
```

> **Reference**: See `references/sql-client.md` for connection options, pool management, savepoints, MySQL specifics, and prepared statement configuration.

## S3 Client (Bun.s3)

Built-in S3 client with Web standard Blob API. Zero dependencies, works with any S3-compatible service (AWS S3, Cloudflare R2, MinIO, etc.). **Use when the project has `AWS_ACCESS_KEY_ID` or S3-compatible credentials in environment.**

```typescript
import { s3, write } from "bun"

// Reads credentials from AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, etc.
const file = s3.file("data.json")              // Lazy reference, no network yet

// Read from S3
const data = await file.json()                  // Download and parse JSON
const text = await file.text()                  // Download as string
const stream = file.stream()                    // ReadableStream

// Upload to S3
await write(s3.file("output.json"), JSON.stringify(data))

// Presigned URLs (synchronous, no network request)
const url = s3.presign("report.pdf", {
  expiresIn: 3600,                              // 1 hour
  method: "PUT",                                // For uploads
  acl: "public-read",
})

// Delete
await file.delete()
```

> **Reference**: See `references/s3-client.md` for custom S3Client, presign options, multipart upload, and serving from Bun.serve.

## Redis Client (Bun.redis)

Built-in Redis/Valkey client with zero dependencies. **Use when the project has `REDIS_URL` or `VALKEY_URL` in environment.**

```typescript
import { redis, RedisClient } from "bun"

// Default client -- reads REDIS_URL from environment
await redis.set("key", "value")
const value = await redis.get("key")            // "value" | null

// With expiration
await redis.set("session", "data", "EX", 3600)

// Counter operations
await redis.incr("counter")
await redis.incrby("counter", 5)

// Hash operations
await redis.hset("user:1", "name", "Alice", "email", "alice@example.com")
await redis.hget("user:1", "name")              // "Alice"

// Custom client
const client = new RedisClient("redis://user:pass@host:6379")
```

> **Reference**: See `references/redis-client.md` for all commands (strings, hashes, lists, sets, sorted sets), pub/sub, pipelines, and common patterns.

## Archive (Bun.Archive)

Create and extract tarballs with optional gzip compression.

```typescript
// Create archive
const archive = new Bun.Archive({
  "hello.txt": "Hello, World!",
  "config.json": JSON.stringify({ key: "value" }),
})
await Bun.write("archive.tar", archive)

// With gzip compression -- write the BYTES, not the Archive (see gotcha below)
const compressed = new Bun.Archive(
  { "hello.txt": "Hello, World!" },
  { compress: "gzip", level: 9 }        // level 1-12, default 6
)
await Bun.write("archive.tar.gz", await compressed.bytes())

// Extract (auto-detects gzip)
const tarball = await Bun.file("archive.tar.gz").bytes()
const extracted = new Bun.Archive(tarball)
await extracted.extract("./out")                   // -> number of entries
await extracted.extract("./out", { glob: ["src/**", "!**/*.test.ts"] })
const files = await extracted.files()              // -> Map<string, File>
```

**Gotcha (verified on v1.4.0):** `Bun.write(path, archive)` ignores the constructor's
`compress` option and writes an uncompressed tar under your `.tar.gz` filename. Bun's own
docs show `Bun.write(path, archive)` as compressing -- it does not. Always pass
`await archive.bytes()` (or `await archive.blob()`), which do honor `compress`.

An `Archive` is **not iterable** -- `for (const [name, contents] of archive)` throws.
Use `await archive.files()` for a `Map<string, File>`, or `await archive.extract(dir)`.

> **Reference**: `node_modules/bun-types/docs/runtime/archive.mdx`

## JSONC (JSON with Comments)

Parse JSON with comments and trailing commas -- replaces `jsonc-parser` or `json5` packages.

```typescript
import { JSONC } from "bun"

const config = JSONC.parse(`{
  // Database config
  "host": "localhost",
  "port": 5432,  // default port
}`)
```

Bun automatically uses JSONC parsing for `tsconfig.json`, `jsconfig.json`, `package.json`, and `bun.lock`. `.jsonc` files can be imported directly: `import config from "./config.jsonc"`.

## Additional Parsing and Utilities (v1.3+)

```typescript
import { JSON5, JSONL, XML, TOML, markdown, cron, secrets } from "bun"

// JSON5 -- superset of JSON (comments, unquoted keys, trailing commas)
const config = JSON5.parse(`{ unquoted: 'value', /* comment */ }`)

// JSONL -- newline-delimited JSON
const records = JSONL.parse('{"a":1}\n{"a":2}\n')
JSONL.parseChunk(partial)                    // { values, read, done, error } for streams

// XML -- SIMD parser + serializer (v1.4+), replaces fast-xml-parser / xml2js
const order = XML.parse('<order id="A1"><item>Tea</item></order>')
// { order: { "@id": "A1", item: "Tea" } }   -- @attr / #text convention, values are strings
XML.parse(doc, { compact: false })           // { name, attributes, children } document tree

// TOML -- rewritten for TOML v1.1.0; stringify() added in v1.4
const cfg = TOML.parse('name = "app"')
TOML.stringify({ name: "app" })

// Markdown -- built-in CommonMark + GFM parser (replaces marked, remark, etc.)
const html = markdown.html("# Title\n\n**Bold** text.")
const ansi = markdown.ansi("# Title")        // ANSI terminal output (v1.3.12+)
markdown.react(readme)                       // React elements (v1.4+)
markdown.render(src, { heading: (c, { level }) => `<h${level}>${c}</h${level}>` })

// Cron -- OS-level jobs, in-process scheduler, and expression parser
const job = cron("0 9 * * 1-5", runReport)   // in-process (v1.3.12+)
const next = cron.parse("0 9 * * 1-5")       // -> Date | null  (NOT a string)

// Secrets -- OS credential store: Keychain / libsecret / Credential Manager (experimental)
await secrets.set({ service: "my-cli", name: "token", value: t })
const token = await secrets.get({ service: "my-cli", name: "token" })  // string | null

// ANSI-aware string utilities (replace wrap-ansi, slice-ansi npm packages)
const coloredText = "\x1b[31mHello, World!\x1b[0m"
Bun.wrapAnsi(coloredText, 80)              // Wrap to column width
Bun.sliceAnsi(coloredText, 0, 5)           // Grapheme-aware slice
```

**Changed in 1.4 -- `Bun.cron` time zone.** `cron.parse()` and the in-process
`cron(schedule, handler)` read schedules in the process's **local** time zone. Before 1.4
they used UTC. Pass `{ tz: "UTC" }` as the final argument to restore the old behavior:

```typescript
cron("0 9 * * *", handler, { tz: "UTC" })
cron.parse("0 9 * * *", Date.now(), { tz: "UTC" })
```

`cron.parse()` returns a `Date`, or `null` when the expression has no match within 8 years
(e.g. February 30th). `cron.remove(title)` takes the **string title** of an OS-level job --
it does not accept a job handle. Stop an in-process job with `job.stop()` or `using`.

**Changed in 1.4 -- stricter parsers.** `TOML.parse()` and `bunfig.toml` now throw
`SyntaxError` on unquoted string values, missing newlines between pairs, and integers past
`Number.MAX_SAFE_INTEGER`. `JSONC.parse()` throws `SyntaxError` on invalid input and on `""`
(it returned `{}` before). `YAML.parse()` follows YAML 1.2, so `yes`/`no`/`on`/`off` are
strings, not booleans -- an `on:` key in a GitHub Actions workflow parses as `"on"`.

> **Reference**: See `references/utilities.md` for full details on all parsing and utility APIs.

---

## SQLite (bun:sqlite)

Built-in SQLite3 with zero dependencies. For **embedded/local databases** -- file-based or in-memory.

```typescript
import { Database } from 'bun:sqlite'

// Open database
const db = new Database('mydb.sqlite')
const db = new Database(':memory:')      // In-memory

// Enable WAL mode (recommended)
db.exec('PRAGMA journal_mode = WAL')

// Execute statements
db.exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)')

// Prepared statements
const insert = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)')
insert.run('Alice', 'alice@example.com')

// Query
const select = db.prepare('SELECT * FROM users WHERE name = ?')
const user = select.get('Alice')         // Single row or null
const users = select.all('Alice')        // All matching rows

// Named parameters
const stmt = db.prepare('SELECT * FROM users WHERE name = $name')
stmt.get({ $name: 'Alice' })

// Transactions
const insertMany = db.transaction((users) => {
  for (const user of users) {
    insert.run(user.name, user.email)
  }
})
insertMany([
  { name: 'Bob', email: 'bob@example.com' },
  { name: 'Carol', email: 'carol@example.com' },
])

// Close
db.close()
```

> **Reference**: See `references/sqlite-and-data.md` for Database constructor, Statement API, transactions, and column types.

## Hashing and Passwords

```typescript
// Non-cryptographic (fast, for hash tables/checksums)
Bun.hash('input')                        // number (wyhash, fastest)
Bun.hash.crc32('input')                  // CRC32

// Cryptographic
new Bun.CryptoHasher('sha256').update('data').digest('hex')

// Password hashing (async, bcrypt by default)
const hash = await Bun.password.hash('password')
const hash = await Bun.password.hash('password', { algorithm: 'argon2id' })
const valid = await Bun.password.verify('password', hash)
```

> **Reference**: See `references/hashing.md` for all hash algorithms, CryptoHasher streaming API, and password hashing options (bcrypt vs argon2id, cost parameters).

## Compression

```typescript
// Gzip
const compressed = Bun.gzipSync(data)         // Uint8Array → Uint8Array
const decompressed = Bun.gunzipSync(compressed)

// Deflate
const compressed = Bun.deflateSync(data)
const decompressed = Bun.inflateSync(compressed)

// Zstandard (zstd)
const compressed = Bun.zstdCompressSync(data)
const decompressed = Bun.zstdDecompressSync(compressed)

// With options
Bun.gzipSync(data, { level: 9, memLevel: 9 })
Bun.deflateSync(data, { level: 6 })
Bun.zstdCompressSync(data, { level: 3 })
```

All compression functions accept `Uint8Array | string | ArrayBuffer` and return `Uint8Array`.

## Utilities

```typescript
// Which (find binary in PATH)
Bun.which('node')                        // '/usr/local/bin/node' or null
Bun.which('bun', { PATH: '/custom/bin' })

// Inspect (like console.log formatting)
Bun.inspect(obj)                         // string
Bun.inspect(obj, { depth: 4, colors: true })

// Module resolution
Bun.resolveSync('./module', '/from/dir')  // Resolved absolute path

// Deep equality
Bun.deepEquals(a, b)                     // boolean (structural equality)
Bun.deepEquals(a, b, true)              // Strict (differentiates 0 and -0)

// Sleep
await Bun.sleep(1000)                    // ms
await Bun.sleep(Bun.nanoseconds() + 1e9) // Until timestamp

// Timing
Bun.nanoseconds()                        // High-resolution timer (bigint)

// UUID
Bun.randomUUIDv7()                       // Time-ordered UUID v7

// String width (for terminal column alignment)
Bun.stringWidth('hello')                 // 5
Bun.stringWidth('你好')                  // 4 (CJK double-width)

// Peek at a promise without awaiting
const value = Bun.peek(promise)          // Returns value if resolved, promise if pending

// Color detection
Bun.color('red', 'css')                  // 'rgb(255, 0, 0)'
Bun.color('#ff0000', 'ansi')             // ANSI escape code
Bun.color('hsl(0, 100%, 50%)', 'number') // 0xff0000
```

> **Reference**: See `references/utilities.md` for complete utility function signatures and examples.

## Semver (Bun.semver)

Built-in semver operations — replaces the `semver` npm package.

```typescript
// Check if a version satisfies a range
Bun.semver.satisfies('1.2.3', '^1.0.0')     // true
Bun.semver.satisfies('2.0.0', '>=1.0 <2.0') // false
Bun.semver.satisfies('1.0.0-beta', '*')      // false (pre-release excluded by default)

// Sort versions (returns -1, 0, or 1)
Bun.semver.order('1.0.0', '2.0.0')          // -1 (a < b)
Bun.semver.order('2.0.0', '1.0.0')          // 1  (a > b)
Bun.semver.order('1.0.0', '1.0.0')          // 0  (equal)

// Sort an array of versions
const versions = ['3.0.0', '1.2.0', '2.1.0']
versions.sort(Bun.semver.order)              // ['1.2.0', '2.1.0', '3.0.0']
```

## Serialization (bun:jsc)

Binary structured clone for efficient serialization.

```typescript
import { serialize, deserialize } from 'bun:jsc'

const data = { key: 'value', nested: [1, 2, 3] }
const bytes = serialize(data)              // Uint8Array
const restored = deserialize(bytes)        // Original structure
```

Faster than `JSON.stringify`/`JSON.parse` for complex objects. Supports types JSON doesn't: `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, etc.

## Image Processing (Bun.Image)

Built-in image decode/transform/encode (v1.3.14+) — replaces `sharp` and `jimp`.

```typescript
const thumb = await Bun.file('upload.jpg')
  .image()
  .resize(400, 400, { fit: 'cover' })
  .rotate(90).flip().flop()
  .modulate({ brightness: 1.1 })
  .webp({ quality: 82 })
  .bytes()

await Bun.file('hero.jpg').image().resize(1024).webp().write('thumb.webp')
const { width, height, format } = await new Bun.Image(buffer).metadata()
const blur = await Bun.file('hero.jpg').image().placeholder()  // thumbhash data URL

const pasted = Bun.Image.fromClipboard()      // v1.4+, macOS/Windows only, null on Linux
```

**Format support is platform-dependent** — do not assume parity:

| | Linux | macOS | Windows |
|---|---|---|---|
| JPEG, PNG, WebP, GIF, BMP | yes | yes | yes |
| HEIC / AVIF | `ERR_IMAGE_FORMAT_UNSUPPORTED` | ImageIO (AVIF **encode** needs Apple Silicon M3+) | WIC + Microsoft Store codec (HEIF Image Extensions / AV1 Video Extension) |
| TIFF decode | no | ImageIO | WIC |
| Clipboard | returns `null` | yes | yes |

JPEG/PNG/WebP use statically-linked codecs, so their encoded output is byte-identical across
platforms; the rest go through the OS backend.

> **Reference**: See `references/image.md`, and
> `node_modules/bun-types/docs/runtime/image.mdx` for the full compatibility matrix.

## Browser Automation (Bun.WebView)

Headless browser automation (v1.3.12+) — navigate, click, type, scroll, run JS, and
screenshot without Playwright or Puppeteer (system WebKit on macOS, or an installed
Chrome/Chromium/Edge on macOS, Linux, and Windows). Clicks and scrolls are real user input.

```typescript
await using view = new Bun.WebView({ width: 1280, height: 720 })
await view.navigate('https://bun.sh')
await view.click("a[href='/docs']")
const title = await view.evaluate('document.title')
await Bun.write('page.png', await view.screenshot())   // returns a Blob
await view.cdp('Page.captureScreenshot', {})           // raw CDP escape hatch
```

> **Reference**: See `references/webview.md`, and
> `node_modules/bun-types/docs/runtime/webview.mdx` for input simulation and CDP events.

## New in Bun 1.4

Compact index — reach for these when the task fits, then read the linked doc before writing code.

| API | Use it for | Doc (`node_modules/bun-types/docs/`) |
|---|---|---|
| `Bun.XML.parse()` / `.stringify()` | XML without `fast-xml-parser`/`xml2js`; `.xml` imports return the parsed doc | `runtime/xml.mdx` |
| `Bun.TOML.stringify()` | Writing TOML (parser now TOML v1.1.0 conformant) | `runtime/toml.mdx` |
| `Bun.secrets` | Storing credentials in the OS keychain instead of a dotfile (experimental) | `runtime/secrets.mdx` |
| `Bun.Terminal` + `Bun.spawn({ terminal })` | Driving `bash`/`vim`/`htop` from JS; replaces `node-pty` | `runtime/child-process.mdx` |
| `Bun.spawn({ cgroup })` | Capping a child's memory/PIDs on Linux before it starts | `runtime/child-process.mdx` |
| `Bun.markdown.react()` | Rendering Markdown to React elements | `runtime/markdown.mdx` |
| `Bun.Image.fromClipboard()` | Reading an image off the system pasteboard (macOS/Windows) | `runtime/image.mdx` |
| `res.textStream()` / `req.textStream()` | Iterating a body as decoded UTF-8 strings, not bytes | `runtime/streams.mdx` |
| `fetch(url, { compress: 'gzip' })` | Compressing a request body and setting `Content-Encoding` | `runtime/networking/fetch.mdx` |
| `fetch(url, { protocol: 'http2' })` | HTTP/2 or HTTP/3 requests (experimental) | `runtime/networking/fetch.mdx` |
| `routes: { '/x/*': { dir: './public' } }` | Serving a directory; replaces `express.static`/`sirv` | `runtime/http/routing.mdx` |
| `process.on('memoryPressure', fn)` | Dropping caches when the OS reports low memory | `runtime/nodejs-compat.mdx` |
| `Bun.isStandaloneExecutable` | Branching inside a `--compile` binary, allocation-free | `bundler/executables.mdx` |
| ML-DSA / ML-KEM | Post-quantum signatures and key encapsulation | `runtime/hashing.mdx` |

`ReadableStream`, `WritableStream`, and `TransformStream` are native as of 1.4 and apply
backpressure automatically — `Bun.serve` pauses a request/response body when the socket
cannot accept more, and `fetch()` pauses the socket when nothing is consuming the body.
Streaming code that previously buffered whole payloads no longer needs hand-rolled
throttling, provided every stage of the pipeline honors backpressure.

> **Reference**: See `references/migration-1.4.md` for behavior that **changed** in 1.4 --
> the one thing Bun's shipped docs do not cover, since they describe only the current state.

## Script Patterns

### CLI Script Template

```typescript
#!/usr/bin/env bun

const args = Bun.argv.slice(2)
const command = args[0]

switch (command) {
  case 'generate':
    await generate(args.slice(1))
    break
  case 'process':
    await process(args.slice(1))
    break
  default:
    console.log('Usage: script <generate|process> [args]')
    process.exit(1)
}
```

### File Generator

```typescript
const glob = new Bun.Glob('**/*.schema.json')

for await (const path of glob.scan('./schemas')) {
  const schema = await Bun.file(`./schemas/${path}`).json()
  const code = generateTypeScript(schema)
  const outPath = path.replace('.schema.json', '.ts')
  await Bun.write(`./generated/${outPath}`, code)
}
```

### Data Pipeline

```typescript
import { $ } from 'bun'
import { Database } from 'bun:sqlite'

// Fetch data
const data = await $`curl -s https://api.example.com/data`.json()

// Process and store
const db = new Database('output.sqlite')
db.exec('CREATE TABLE IF NOT EXISTS items (id TEXT PRIMARY KEY, value TEXT)')

const insert = db.prepare('INSERT OR REPLACE INTO items (id, value) VALUES (?, ?)')
const batch = db.transaction((items) => {
  for (const item of items) {
    insert.run(item.id, JSON.stringify(item))
  }
})

batch(data.items)
db.close()
```

### Best Practices

1. **Prefer `Bun.file()` + `Bun.write()`** over `fs.readFile`/`fs.writeFile`
2. **Use `Bun.$`** for shell commands instead of `child_process`
3. **Use `Bun.sql()`** for PostgreSQL/MySQL when `DATABASE_URL` is available -- zero-dependency, connection pooling, tagged templates
4. **Use `bun:sqlite`** for embedded/local SQLite databases instead of external packages
5. **Use `Bun.Glob`** instead of `glob` npm package
6. **Use `Bun.CryptoHasher`** instead of `crypto.createHash`
7. **Use `Bun.password`** instead of `bcrypt`/`argon2` npm packages
8. **Use `Bun.gzipSync`/`Bun.zstdCompressSync`** instead of `zlib`
9. **Use `Bun.env`** for environment variables (same as `process.env` but typed)
10. **Use `import.meta.dir`** instead of `__dirname` (or `import.meta.dirname` for Node compat)
11. **Use `Bun.which()`** instead of `which` npm package
12. **Use `Bun.s3`** instead of `@aws-sdk/client-s3` for S3 operations
13. **Use `Bun.redis`** instead of `ioredis` or `redis` npm packages
14. **Use `Bun.Archive`** instead of `tar` or `archiver` npm packages for tarballs
15. **Use `JSONC.parse()`** instead of `jsonc-parser` package
16. **Use `JSON5.parse()`** instead of `json5` package
17. **Use `JSONL.parse()`** instead of manual newline splitting for JSON Lines
18. **Use `markdown.html()`/`markdown.ansi()`** instead of `marked`, `remark`, or `markdown-it` packages
19. **Use `Bun.wrapAnsi()`** instead of `wrap-ansi` npm package
20. **Use `Bun.sliceAnsi()`** instead of `slice-ansi` npm package
21. **Use `Bun.Image`** instead of `sharp` or `jimp` for image processing
22. **Use `Bun.XML`** instead of `fast-xml-parser` or `xml2js` (v1.4+)
23. **Use `Bun.secrets`** instead of writing credentials to a dotfile (v1.4+)
24. **Use `Bun.Terminal`** instead of `node-pty` for pseudo-terminals
25. **Use `URLPattern`** instead of `path-to-regexp`
26. **Use `CompressionStream`/`DecompressionStream`** for streaming compression
27. **Read `node_modules/bun-types/docs/**/*.mdx`** before writing non-trivial Bun code --
    open by explicit path, and never edit anything under `node_modules/`

## References

> - `references/migration-1.4.md` -- what changed between 1.3 and 1.4 (breaking behavior)
> - `references/http-server.md` -- `Bun.serve` routes, TLS, WebSockets, static files
> - `references/file-io.md`, `references/shell-and-process.md`, `references/networking.md`
> - `references/sql-client.md`, `references/sqlite-and-data.md`, `references/redis-client.md`, `references/s3-client.md`
> - `references/utilities.md`, `references/hashing.md`, `references/image.md`, `references/webview.md`
