# File I/O Reference

## Bun.file()

Creates a lazy `BunFile` reference. No I/O happens until you call a read method.

```typescript
Bun.file(path: string | URL): BunFile
Bun.file(fd: number): BunFile        // From file descriptor
```

### BunFile Interface

```typescript
interface BunFile extends Blob {
  // Read methods (all async)
  text(): Promise<string>
  json(): Promise<any>
  arrayBuffer(): Promise<ArrayBuffer>
  stream(): ReadableStream<Uint8Array>
  blob(): Promise<Blob>
  bytes(): Promise<Uint8Array>

  // Metadata
  readonly size: number              // File size in bytes
  readonly type: string              // MIME type (auto-detected from extension)
  readonly name: string | undefined  // File path

  // Existence check
  exists(): Promise<boolean>

  // Writer (for incremental writes)
  writer(options?: { highWaterMark?: number }): FileSink

  // Last modified
  readonly lastModified: number      // Unix timestamp (ms)
}
```

### MIME Type Detection

Bun auto-detects MIME types from file extensions:
- `.json` → `application/json`
- `.txt` → `text/plain`
- `.html` → `text/html`
- `.css` → `text/css`
- `.js` → `text/javascript`
- `.ts` → `application/typescript`
- `.png` → `image/png`
- `.jpg`/`.jpeg` → `image/jpeg`
- `.svg` → `image/svg+xml`
- `.pdf` → `application/pdf`
- `.wasm` → `application/wasm`

## Bun.write()

Write data to a file or file descriptor. Creates parent directories if needed.

```typescript
Bun.write(
  destination: string | BunFile | FileBlob | number,  // Path, BunFile, or fd
  input: string | Blob | BunFile | ArrayBuffer | Uint8Array | Response | ReadableStream,
  options?: { createPath?: boolean }  // createPath: true by default
): Promise<number>  // Returns bytes written
```

### Overload Examples

```typescript
// String to path
await Bun.write('output.txt', 'content')

// BunFile to path (efficient copy, uses sendfile/copy_file_range)
await Bun.write('copy.txt', Bun.file('original.txt'))

// ArrayBuffer/Uint8Array to path
await Bun.write('binary.dat', new Uint8Array([0x00, 0x01, 0x02]))

// Response body to path
const response = await fetch('https://example.com/image.png')
await Bun.write('image.png', response)

// String to BunFile
await Bun.write(Bun.file('output.txt'), 'content')

// String to file descriptor
await Bun.write(1, 'to stdout')     // fd 1 = stdout
await Bun.write(2, 'to stderr')     // fd 2 = stderr

// ReadableStream to file
await Bun.write('stream-output.txt', readableStream)
```

## FileSink (Incremental Writer)

For writing large files incrementally without holding everything in memory.

```typescript
const file = Bun.file('large-output.txt')
const writer = file.writer({ highWaterMark: 1024 * 1024 }) // 1MB buffer

writer.write('line 1\n')
writer.write('line 2\n')
writer.write(new Uint8Array([0x0a]))
writer.flush()               // Force flush buffer to disk
writer.end()                 // Flush and close
```

### FileSink Interface

```typescript
interface FileSink {
  write(data: string | Uint8Array | ArrayBuffer): number  // Returns bytes written
  flush(): void              // Flush buffer to disk
  end(data?: string | Uint8Array): number  // Write final data, flush, close
  start(options?: { highWaterMark?: number }): void
  ref(): void                // Prevent process exit
  unref(): void              // Allow process exit
}
```

## Stdio

```typescript
Bun.stdin   // BunFile for standard input
Bun.stdout  // BunFile for standard output
Bun.stderr  // BunFile for standard error
```

### Reading stdin

```typescript
// Read all at once
const allInput = await Bun.stdin.text()

// Read as JSON
const data = await Bun.stdin.json()

// Read as bytes
const bytes = await Bun.stdin.bytes()

// Stream stdin
const reader = Bun.stdin.stream().getReader()
while (true) {
  const { done, value } = await reader.read()
  if (done) break
  // process value (Uint8Array)
}

// Async iteration
for await (const chunk of Bun.stdin.stream()) {
  // process chunk
}
```

### Writing to stdout/stderr

```typescript
await Bun.write(Bun.stdout, 'Hello stdout\n')
await Bun.write(Bun.stderr, 'Hello stderr\n')

// Or use the writer for multiple writes
const out = Bun.stdout.writer()
out.write('line 1\n')
out.write('line 2\n')
out.flush()
```

## File Watching

```typescript
import { watch } from 'fs'

// Watch a file
const watcher = watch('file.txt', (event, filename) => {
  console.log(event, filename)   // 'change' or 'rename'
})

// Watch a directory
const watcher = watch('./src', { recursive: true }, (event, filename) => {
  console.log(event, filename)
})

// Stop watching
watcher.close()
```

Note: Bun uses native OS file watching (FSEvents on macOS, inotify on Linux).

## ReadableStream Utilities

```typescript
// Convert ReadableStream to string
const text = await new Response(stream).text()

// Convert ReadableStream to JSON
const json = await new Response(stream).json()

// Convert ReadableStream to ArrayBuffer
const buf = await new Response(stream).arrayBuffer()

// Pipe ReadableStream to file
await Bun.write('output.txt', stream)
```

## Temporary Files

Bun doesn't have a built-in temp file API, but you can use:

```typescript
import { tmpdir } from 'os'
import { join } from 'path'

const tmpPath = join(tmpdir(), `bun-${Bun.randomUUIDv7()}`)
await Bun.write(tmpPath, 'temp content')

// Clean up
const { unlinkSync } = require('fs')
unlinkSync(tmpPath)
```
