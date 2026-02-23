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
