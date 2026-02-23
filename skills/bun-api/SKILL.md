---
name: bun-api
description: Bun runtime API reference for TypeScript scripts. Covers Bun.file(), Bun.write(), Bun.$() shell, Bun.spawn(), Bun.Glob, Bun.env, bun:sqlite, Bun.hash, Bun.password, compression, and utilities for file generation, data processing, and scripting.
---

# Bun Runtime API

Use Bun's native APIs instead of Node.js equivalents when writing TypeScript scripts for Bun projects. Bun APIs are faster, more ergonomic, and require no additional dependencies.

**Critical**: In a Bun project (has `bun.lock`, `bun.lockb`, `bunfig.toml`, or `@types/bun` in devDependencies), always use Bun to run scripts (`bun file.ts`, not `node file.ts`) and prefer Bun-native APIs over Node.js equivalents. Mixing runtimes causes subtle bugs and unnecessary retries.

## When to Use

- Scripts for generating files, parsing data, running migrations
- File processing and transformation pipelines
- Shell scripting and automation
- Database operations with SQLite
- Any scripting task in a Bun project

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

## SQLite (bun:sqlite)

Built-in SQLite3 with zero dependencies.

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

### Non-Cryptographic Hashing

```typescript
Bun.hash('input')                        // number (wyhash, fast)
Bun.hash('input', seed)                  // With seed
Bun.hash.crc32('input')                  // CRC32
Bun.hash.adler32('input')               // Adler-32
Bun.hash.cityHash32('input')            // CityHash32
Bun.hash.cityHash64('input')            // CityHash64 (bigint)
Bun.hash.murmur32v3('input')            // MurmurHash3 32-bit
Bun.hash.murmur64v2('input')            // MurmurHash3 64-bit (bigint)
```

### Cryptographic Hashing

```typescript
// One-shot
new Bun.CryptoHasher('sha256').update('data').digest('hex')

// Streaming
const hasher = new Bun.CryptoHasher('sha256')
hasher.update('part1')
hasher.update('part2')
const hash = hasher.digest('hex')        // or 'base64', Uint8Array (no arg)

// Algorithms: 'sha1', 'sha256', 'sha384', 'sha512', 'md5', 'blake2b256', etc.
```

### Password Hashing

```typescript
// Hash password (async, uses bcrypt by default)
const hash = await Bun.password.hash('password')
const hash = await Bun.password.hash('password', { algorithm: 'argon2id' })

// Verify
const valid = await Bun.password.verify('password', hash)      // true/false

// Sync versions
const hash = Bun.password.hashSync('password')
const valid = Bun.password.verifySync('password', hash)
```

> **Reference**: See `references/sqlite-and-data.md` for all hash algorithms and CryptoHasher details.

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

## Serialization (bun:jsc)

Binary structured clone for efficient serialization.

```typescript
import { serialize, deserialize } from 'bun:jsc'

const data = { key: 'value', nested: [1, 2, 3] }
const bytes = serialize(data)              // Uint8Array
const restored = deserialize(bytes)        // Original structure
```

Faster than `JSON.stringify`/`JSON.parse` for complex objects. Supports types JSON doesn't: `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, etc.

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
3. **Use `bun:sqlite`** instead of external SQLite packages
4. **Use `Bun.Glob`** instead of `glob` npm package
5. **Use `Bun.CryptoHasher`** instead of `crypto.createHash`
6. **Use `Bun.password`** instead of `bcrypt`/`argon2` npm packages
7. **Use `Bun.gzipSync`/`Bun.zstdCompressSync`** instead of `zlib`
8. **Use `Bun.env`** for environment variables (same as `process.env` but typed)
9. **Use `import.meta.dir`** instead of `__dirname` (or `import.meta.dirname` for Node compat)
10. **Use `Bun.which()`** instead of `which` npm package
