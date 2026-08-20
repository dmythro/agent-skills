# Utilities Reference

## Bun.which()

Find the path to an executable in PATH.

```typescript
Bun.which(command: string): string | null
Bun.which(command: string, options: { PATH?: string; cwd?: string }): string | null
```

```typescript
Bun.which('node')         // '/usr/local/bin/node'
Bun.which('nonexistent')  // null

// Custom PATH
Bun.which('tool', { PATH: '/custom/bin:/usr/bin' })

// Search relative to cwd too
Bun.which('./local-tool', { cwd: '/project' })
```

## Bun.inspect()

Format a value as a human-readable string (like `console.log` but returns a string).

```typescript
Bun.inspect(value: any): string
Bun.inspect(value: any, options?: InspectOptions): string
```

```typescript
interface InspectOptions {
  depth?: number              // Max nesting depth (default: 2)
  colors?: boolean            // ANSI colors (default: false)
  sorted?: boolean            // Sort object keys
  compact?: boolean           // Compact output
}
```

```typescript
Bun.inspect({ a: 1, b: [2, 3] })
// '{ a: 1, b: [ 2, 3 ] }'

Bun.inspect(obj, { depth: 10, colors: true })
```

## Bun.resolveSync()

Resolve a module specifier to an absolute file path using Bun's module resolution.

```typescript
Bun.resolveSync(specifier: string, parent: string): string
```

```typescript
Bun.resolveSync('./utils', '/project/src')
// '/project/src/utils.ts'

Bun.resolveSync('react', '/project/src')
// '/project/node_modules/react/index.js'

// Throws if module not found
try {
  Bun.resolveSync('nonexistent-pkg', import.meta.dir)
} catch (e) {
  // ResolveMessage: Module not found
}
```

## Bun.deepEquals()

Structural deep equality comparison.

```typescript
Bun.deepEquals(a: any, b: any): boolean
Bun.deepEquals(a: any, b: any, strict: boolean): boolean
```

```typescript
Bun.deepEquals({ a: 1 }, { a: 1 })          // true
Bun.deepEquals([1, 2], [1, 2])               // true
Bun.deepEquals(new Map([['a', 1]]), new Map([['a', 1]]))  // true

// Strict mode differentiates:
Bun.deepEquals(0, -0)                        // true
Bun.deepEquals(0, -0, true)                  // false (strict)
```

## Bun.sleep()

Async sleep.

```typescript
Bun.sleep(ms: number): Promise<void>
```

```typescript
await Bun.sleep(1000)    // Sleep 1 second
await Bun.sleep(0)       // Yield to event loop
```

## Bun.nanoseconds()

High-resolution timer.

```typescript
Bun.nanoseconds(): number   // Nanoseconds since Bun process started
```

```typescript
const start = Bun.nanoseconds()
// ... operation ...
const elapsed = Bun.nanoseconds() - start
console.log(`Took ${elapsed / 1e6}ms`)
```

## Bun.randomUUIDv7()

Generate a time-ordered UUID v7 (sortable by creation time).

```typescript
Bun.randomUUIDv7(): string
```

```typescript
Bun.randomUUIDv7()  // '0192e4a2-7b3c-7def-8a12-3456789abcde'
```

Prefer v7 over v4 for database primary keys (monotonically increasing, better index performance).

## Bun.stringWidth()

Calculate the display width of a string in terminal columns. Handles CJK double-width characters, emoji, ANSI escapes, etc.

```typescript
Bun.stringWidth(str: string): number
Bun.stringWidth(str: string, options?: { countAnsiEscapeCodes?: boolean }): number
```

```typescript
Bun.stringWidth('hello')        // 5
Bun.stringWidth('你好')         // 4 (CJK chars are double-width)
Bun.stringWidth('🎉')           // 2 (emoji is double-width)
Bun.stringWidth('\x1b[31mred\x1b[0m')  // 3 (ANSI escapes not counted)
Bun.stringWidth('\x1b[31mred\x1b[0m', { countAnsiEscapeCodes: true })  // 11
```

## Bun.peek()

Synchronously read a promise's value if already resolved, without awaiting.

```typescript
Bun.peek(promise: Promise<T>): T | Promise<T>
Bun.peek.status(promise: Promise<T>): 'fulfilled' | 'rejected' | 'pending'
```

```typescript
const p = Promise.resolve(42)
Bun.peek(p)           // 42 (synchronous!)

const pending = new Promise(() => {})
Bun.peek(pending)     // Returns the promise itself (still pending)

Bun.peek.status(p)        // 'fulfilled'
Bun.peek.status(pending)  // 'pending'
```

## Bun.color()

Parse and convert CSS colors between formats.

```typescript
Bun.color(input: string, outputFormat: 'css' | 'ansi' | 'ansi-16' | 'ansi-256' | 'ansi-16m' | 'number' | 'rgb' | 'rgba' | '{rgba}' | '[rgba]' | 'hex' | 'HEX'): string | number | [number, number, number, number] | null
```

```typescript
Bun.color('red', 'css')           // 'rgb(255, 0, 0)'
Bun.color('#ff0000', 'number')    // 16711680 (0xFF0000)
Bun.color('rgb(255,0,0)', 'hex')  // '#ff0000'
Bun.color('hsl(0, 100%, 50%)', '[rgba]')  // [255, 0, 0, 255]
Bun.color('invalid', 'css')       // null
```

## JSONC

Parse JSON with comments and trailing commas. Available as a named import from `"bun"`.

```typescript
import { JSONC } from "bun"

// Parse JSONC string
const config = JSONC.parse(`{
  // Database settings
  "host": "localhost",
  "port": 5432,  // default Postgres port
}`)
// { host: "localhost", port: 5432 }
```

Bun automatically uses the JSONC loader for `tsconfig.json`, `jsconfig.json`, `package.json`, and `bun.lock` files. You can also import `.jsonc` files directly:

```typescript
import config from "./config.jsonc"
```

## Bun.Archive

Create and extract tarballs with optional gzip compression.

```typescript
// Create an archive from an object mapping filenames to content
const archive = new Bun.Archive({
  "hello.txt": "Hello, World!",
  "data.json": JSON.stringify({ key: "value" }),
})

// Write uncompressed tar
await Bun.write("archive.tar", archive)

// Create with gzip compression -- write the bytes, not the Archive (see below)
const compressed = new Bun.Archive(
  { "hello.txt": "Hello, World!" },
  { compress: "gzip" }
)
await Bun.write("archive.tar.gz", await compressed.bytes())

// Gzip with custom compression level (1-12, default 6)
const maxCompressed = new Bun.Archive(
  { "hello.txt": "Hello, World!" },
  { compress: "gzip", level: 12 }
)
```

File contents may be `string`, `Blob` (including `Bun.file()`), `ArrayBufferView`, or `ArrayBuffer`.

### Compression Gotcha

**`Bun.write(path, archive)` ignores the constructor's `compress` option** and writes a plain
tar, even when the filename ends in `.tar.gz`. Verified on v1.4.0: the output carries a POSIX
tar header, not the gzip magic bytes. Bun's own docs show this form as compressing -- it does
not (tracked upstream: oven-sh/bun#30234). `.bytes()` and `.blob()` do honor `compress`:

```typescript
await Bun.write("out.tar.gz", archive)                 // WRONG -- uncompressed tar
await Bun.write("out.tar.gz", await archive.bytes())   // correct -- real gzip
```

### Reading Archives

An `Archive` is **not iterable** -- `for (const [name, content] of archive)` throws
`TypeError: {} is not iterable`. Use `files()` or `extract()`.

```typescript
// Read from file (auto-detects gzip)
const tarball = await Bun.file("archive.tar.gz").bytes()
const archive = new Bun.Archive(tarball)

// Read contents into memory -- Map<string, File>
const files = await archive.files()
for (const [path, file] of files) {
  console.log(path, file.size, await file.text())
}

// Filter with globs (negative patterns supported)
const tsFiles = await archive.files(["**/*.ts", "!**/*.test.ts"])

// Extract straight to disk -- returns the number of entries written
const count = await archive.extract("./output")
await archive.extract("./output", { glob: ["src/**", "!node_modules/**"] })
```

`extract()` creates the target directory, rejects absolute paths and unsafe symlink targets,
and normalizes away `..` traversal. Windows always skips symlinks. `files()` loads contents
into memory and returns regular files only -- prefer `extract()` for large archives.

> **Reference**: `node_modules/bun-types/docs/runtime/archive.mdx`

## Bun.markdown

Built-in CommonMark-compliant Markdown parser -- replaces `marked`, `markdown-it`, or `remark`. `markdown` is a namespace (`markdown.html`, `markdown.ansi`, `markdown.render`, `markdown.react`), available as `Bun.markdown` or a named import from `"bun"`.

```typescript
import { markdown } from "bun"

// Render to HTML
const html = markdown.html("# Hello\n\nThis is **bold** text.")
// '<h1>Hello</h1>\n<p>This is <strong>bold</strong> text.</p>\n'

// Render to ANSI-colored terminal output (v1.3.12+)
process.stdout.write(markdown.ansi("# Hello\n\n**bold** and *italic*\n"))
```

### markdown.ansi() Options (v1.3.12+)

```typescript
markdown.ansi("# Hi", { colors: false })                       // plain text, no ANSI
markdown.ansi("[docs](https://bun.sh)", { hyperlinks: true })  // OSC 8 terminal links
markdown.ansi(longText, { columns: 60 })                       // wrap to width
markdown.ansi("![alt](./logo.png)", { kittyGraphics: true })   // inline images (kitty)
```

| Option | Description |
|---|---|
| `colors` | Emit ANSI color codes (default: true) |
| `hyperlinks` | Emit OSC 8 terminal hyperlinks |
| `columns` | Wrap output to a column width |
| `kittyGraphics` | Render images inline via the kitty graphics protocol |

## JSON5

Parse JSON5 format (superset of JSON with comments, trailing commas, unquoted keys, etc.) -- replaces the `json5` npm package. Available as a named import from `"bun"`.

```typescript
import { JSON5 } from "bun"

const config = JSON5.parse(`{
  // Comments allowed
  unquoted: 'keys work',
  trailing: 'commas',
}`)
```

## JSONL

Parse and produce JSON Lines (newline-delimited JSON) format. Available as a named import from `"bun"`.

```typescript
import { JSONL } from "bun"

// Parse JSONL string
const records = JSONL.parse('{"a":1}\n{"a":2}\n{"a":3}')
// [{ a: 1 }, { a: 2 }, { a: 3 }]
```

## cron / Bun.cron

Cron scheduler and expression parser. The named `cron` import is the same value as `Bun.cron`.
Schedules use 5 fields (`minute hour day month weekday`) -- seconds are not supported. Named
months/weekdays (`MON-FRI`, `JAN`) and nicknames (`@daily`, `@hourly`) work.

`Bun.cron` has two distinct scheduling forms plus a parser. Pick by whether the job must
survive process exit:

| | In-process `cron(schedule, handler)` | OS-level `cron(path, schedule, title)` |
|---|---|---|
| Survives exit/reboot | No | Yes |
| Shared state between runs | Yes | No -- fresh process each time |
| Requires | Nothing | crontab / launchd / Task Scheduler |
| Returns | `CronJob` (sync) | `Promise<void>` |

### In-Process Scheduling (v1.3.12+)

```typescript
import { cron } from "bun"

// Run a callback on a schedule (in-process -- no external cron daemon)
const job = cron("0 9 * * 1-5", async () => {
  await sendDailyReport()          // weekdays at 09:00 LOCAL time (see below)
})

job.cron          // "0 9 * * 1-5"
job.stop()        // stop the job
job.unref()       // don't keep the process alive
job.ref()         // keep the process alive (default)

// Auto-stop at scope exit with explicit resource management
{
  using daily = cron("0 0 * * *", rotateLogs)
}  // job disposed here
```

Runs never overlap: Bun computes the next fire time only after the handler (and any promise
it returns) settles, so a handler that overruns its interval delays the next fire rather than
stacking. A synchronous `throw` surfaces as `process.on("uncaughtException")`; a rejected
promise as `process.on("unhandledRejection")`. With a listener installed the job keeps
running. Under `bun --hot`, jobs are stopped before the module graph re-evaluates and
re-registered from source, so edits do not leak timers. `jest.useFakeTimers()` drives them.

### OS-Level Jobs (v1.3.11+)

Registers a real crontab/launchd/Task Scheduler entry that runs a script on a schedule. The
script exports a `scheduled()` handler, matching Cloudflare Workers Cron Triggers.

```typescript
await Bun.cron("./worker.ts", "30 2 * * MON", "weekly-report")
await Bun.cron.remove("weekly-report")     // takes the TITLE string, not a job handle

// worker.ts
export default {
  async scheduled(controller: Bun.CronController) {
    controller.cron           // "30 2 * * 1"
    controller.scheduledTime  // epoch ms at invocation
    await doWork()
  },
}
```

Re-registering the same `title` replaces the existing job in place. Removing a job that does
not exist resolves without error.

### Parsing

```typescript
cron.parse("0 9 * * 1-5")                        // -> Date (next match) | null
cron.parse("0 * * * *", cursor)                  // search from a Date | epoch ms
cron.parse("0 9 * * *", Date.now(), { tz: "UTC" })
cron.parse("0 0 30 2 *")                         // null -- no match within 8 years
```

`parse()` returns a **`Date`**, not a string. It returns `null` when the expression can never
match (February 30th, for example).

### Time Zone -- Changed in 1.4

**On v1.4+**, `cron.parse()` and the in-process `cron(schedule, handler)` read schedules in
the process's **local** time zone, matching the OS-level form. **On v1.3.x they used UTC.**
Both accept `{ tz }` as a final argument:

```typescript
cron("0 9 * * *", handler, { tz: "UTC" })              // pre-1.4 behavior on 1.4+
cron.parse("0 9 * * *", Date.now(), { tz: "America/New_York" })
```

Code written against 1.3 that assumed UTC will fire at a different wall-clock time after
upgrading unless `{ tz: "UTC" }` is added.

When both day-of-month and day-of-week are set (neither is `*`), the expression matches when
**either** is true, per POSIX cron.

> **Reference**: `node_modules/bun-types/docs/runtime/cron.mdx` -- DST handling, per-platform
> registration details, and log locations.

## Bun.secrets (v1.4+)

Store credentials in the OS credential manager instead of a plaintext dotfile: Keychain on
macOS, libsecret on Linux, Credential Manager on Windows. Experimental -- the API may change.

```typescript
import { secrets } from "bun"

await secrets.set({ service: "my-cli", name: "github-token", value: token })
const token = await secrets.get({ service: "my-cli", name: "github-token" })  // string | null
const removed = await secrets.delete({ service: "my-cli", name: "github-token" })  // boolean
```

Use the options-object form. Bun's prose docs also show a positional
`secrets.get("service", "name")`, but the 1.4.0 type definitions declare only
`get(options)`, so the positional call fails typecheck.

All operations are async and run on Bun's threadpool. Intended for local development tools,
not production deployment secrets.

> **Reference**: `node_modules/bun-types/docs/runtime/secrets.mdx`

## Bun.XML (v1.4+)

SIMD XML parser and serializer -- replaces `fast-xml-parser` and `xml2js`. `.xml` files can be
imported directly in both the runtime and the bundler.

```typescript
import { XML } from "bun"

// Compact shape (default): @attr / #text convention
XML.parse('<order id="A1"><item>Tea</item><item>Mug</item><paid/></order>')
// { order: { "@id": "A1", item: ["Tea", "Mug"], paid: "" } }

// Document tree -- preserves order, comments, processing instructions
XML.parse("<p>Hi <b>you</b></p>", { compact: false })
// { name: "p", attributes: {}, children: ["Hi ", { name: "b", ... }] }

XML.stringify({ order: { "@id": "A1", item: "Tea" } })
XML.parse(await Bun.file("feed.xml").bytes())   // bytes honor BOM / encoding declaration
```

Every value is a **string** -- nothing is coerced to number, boolean, or `null`. In the compact
shape a repeated child name becomes an array and a single one does not, so read defensively:

```typescript
const entries = [feed.entry ?? []].flat()      // an array either way
for (const e of entries) {
  const title = typeof e.title === "string" ? e.title : (e.title?.["#text"] ?? "")
}
```

`parse()` throws `SyntaxError` on malformed input (no lenient mode) and `RangeError` on
pathological nesting.

**Changed in 1.4:** importing a `.xml` file now returns the parsed document. Before, it
returned the file's path. Pass `--loader .xml:file` to get the path back.

> **Reference**: `node_modules/bun-types/docs/runtime/xml.mdx`

## Explicit Resource Management (using / await using)

Bun natively supports the TC39 `using` and `await using` declarations (v1.3.12+; emitted without transpilation when targeting Bun as of v1.3.14). A value with `Symbol.dispose` or `Symbol.asyncDispose` is released automatically at scope exit.

```typescript
{
  using job = Bun.cron('0 * * * *', hourly)   // job.stop() runs at scope exit
}

{
  await using view = new Bun.WebView()         // view.close() runs at scope exit
  await view.navigate('https://bun.sh')
}
```

`Bun.cron()` jobs and `Bun.WebView` are common examples of disposable Bun resources.

## Bun.wrapAnsi() / Bun.sliceAnsi()

ANSI-aware string manipulation for terminal output.

```typescript
const coloredText = "\x1b[31mHello, World!\x1b[0m"

// Wrap text with ANSI codes to a column width
Bun.wrapAnsi(coloredText, 80)          // 33-88x faster than wrap-ansi npm

// Slice a string while preserving ANSI escape codes
Bun.sliceAnsi(coloredText, 0, 5)       // Grapheme/ANSI-aware slicing
```

## Compression Functions

### Gzip

```typescript
Bun.gzipSync(data: Uint8Array | string | ArrayBuffer, options?: { level?: number; memLevel?: number; windowBits?: number }): Uint8Array
Bun.gunzipSync(data: Uint8Array | ArrayBuffer): Uint8Array
```

```typescript
const compressed = Bun.gzipSync('hello world')
const original = Bun.gunzipSync(compressed)
new TextDecoder().decode(original)  // 'hello world'

// Max compression
Bun.gzipSync(data, { level: 9 })
```

### Deflate

```typescript
Bun.deflateSync(data: Uint8Array | string | ArrayBuffer, options?: { level?: number; memLevel?: number; windowBits?: number }): Uint8Array
Bun.inflateSync(data: Uint8Array | ArrayBuffer): Uint8Array
```

### Zstandard (zstd)

```typescript
Bun.zstdCompressSync(data: Uint8Array | string | ArrayBuffer, options?: { level?: number }): Uint8Array
Bun.zstdDecompressSync(data: Uint8Array | ArrayBuffer, knownSize?: number): Uint8Array
```

```typescript
// Fast compression (level 1)
Bun.zstdCompressSync(data, { level: 1 })

// Better compression (level 19)
Bun.zstdCompressSync(data, { level: 19 })

// Default level: 3
Bun.zstdCompressSync(data)
```

## Bun.Glob

Pattern matching for file paths.

```typescript
new Bun.Glob(pattern: string)
```

### Methods

```typescript
interface Glob {
  // Test if a string matches the pattern
  match(path: string): boolean

  // Async scan directory
  scan(options?: string | GlobScanOptions): AsyncIterable<string>

  // Sync scan directory
  scanSync(options?: string | GlobScanOptions): Iterable<string>
}

interface GlobScanOptions {
  cwd?: string              // Directory to scan (default: '.')
  dot?: boolean             // Include dotfiles (default: false)
  onlyFiles?: boolean       // Skip directories (default: true)
  absolute?: boolean        // Return absolute paths (default: false)
  followSymlinks?: boolean  // Follow symlinks (default: false)
}
```

### Pattern Syntax

| Pattern | Matches |
|---|---|
| `*` | Any characters except `/` |
| `**` | Any characters including `/` (recursive) |
| `?` | Single character except `/` |
| `[abc]` | Character class |
| `[!abc]` or `[^abc]` | Negated character class |
| `{a,b,c}` | Alternation |

### Examples

```typescript
const glob = new Bun.Glob('**/*.{ts,tsx}')

// Check match
glob.match('src/index.ts')      // true
glob.match('README.md')         // false

// Scan files
for await (const path of glob.scan('./src')) {
  console.log(path)             // Relative paths
}

// Scan with options
const paths = []
for await (const path of glob.scan({
  cwd: './src',
  absolute: true,
  dot: true,
  followSymlinks: true,
})) {
  paths.push(path)
}

// Collect all matches into array
const allFiles = Array.from(glob.scanSync('./src'))
```

## Bun.env

Typed environment variable access (alias for `process.env`).

```typescript
Bun.env.NODE_ENV       // string | undefined
Bun.env.DATABASE_URL   // string | undefined
Bun.env.PORT           // string | undefined

// Set
Bun.env.MY_VAR = 'value'

// Delete
delete Bun.env.MY_VAR
```

## Bun.argv

Command-line arguments.

```typescript
Bun.argv: string[]
// [0] = path to bun executable
// [1] = path to script
// [2+] = user arguments
```

```typescript
// script.ts called as: bun script.ts --flag value
Bun.argv[0]  // '/usr/local/bin/bun'
Bun.argv[1]  // '/path/to/script.ts'
Bun.argv[2]  // '--flag'
Bun.argv[3]  // 'value'
```

## Bun.main

The absolute path to the entry point script.

```typescript
Bun.main: string  // '/path/to/entry.ts'
```

Useful for determining if a file is the entry point:
```typescript
if (import.meta.path === Bun.main) {
  // This file is the entry point
  main()
}
```

## import.meta Properties

```typescript
import.meta.dir       // Directory of current file (no trailing slash)
import.meta.file      // Filename (basename) of current file
import.meta.path      // Full absolute path of current file
import.meta.dirname   // Same as import.meta.dir (Node.js compat)
import.meta.filename  // Same as import.meta.path (Node.js compat)
import.meta.url       // file:// URL of current file
import.meta.main      // boolean: true if this is the entry point
import.meta.resolve(specifier)  // Resolve module specifier to URL string
```
