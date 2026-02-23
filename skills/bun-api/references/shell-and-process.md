# Shell and Process Reference

## Bun.$ (Shell API)

Tagged template literal for shell commands with automatic escaping.

```typescript
import { $ } from 'bun'
```

### Basic Usage

```typescript
const result = await $`command arg1 arg2`
```

### ShellOutput Interface

```typescript
interface ShellOutput {
  readonly exitCode: number
  readonly stderr: Buffer
  readonly stdout: Buffer

  text(): string               // stdout as UTF-8 string
  json(): any                  // Parse stdout as JSON
  lines(): string[]            // Split stdout on newlines
  bytes(): Uint8Array          // stdout as Uint8Array
  blob(): Blob                 // stdout as Blob
}
```

### Interpolation and Escaping

Variables are automatically escaped to prevent injection:

```typescript
const filename = 'file with spaces.txt'
await $`cat ${filename}`                // Safe: properly quoted

const files = ['a.txt', 'b.txt', 'c.txt']
await $`cat ${files}`                   // Arrays are expanded with proper escaping

// Template expressions (NOT escaped — use for building command parts)
const flags = '--verbose --color'
await $`ls ${$.raw(flags)}`             // Raw insertion (careful!)
```

### Modifiers

```typescript
// Quiet mode — suppress stdout printing
await $`npm install`.quiet()

// No-throw — don't throw on non-zero exit code
const result = await $`may-fail`.nothrow()
if (result.exitCode !== 0) { /* handle */ }

// Combined
await $`risky-command`.quiet().nothrow()

// Environment variables
await $`echo $MY_VAR`.env({ MY_VAR: 'hello' })

// Working directory
await $`ls`.cwd('/tmp')

// Timeout (ms)
await $`slow-command`.timeout(5000)

// Stdin
await $`cat`.stdin('input text')
await $`cat`.stdin(Buffer.from('bytes'))
await $`cat`.stdin(Bun.file('input.txt'))
```

### Piping

```typescript
// Shell pipes
await $`cat file.txt | grep pattern | sort | uniq`

// Programmatic piping between commands
const result = await $`echo "hello world"`.pipe($`grep hello`)

// Pipe to file
await $`echo hello`.redirect('output.txt')
```

### Redirects

```typescript
// stdout to file
await $`echo hello > output.txt`

// Append
await $`echo hello >> output.txt`

// stderr to file
await $`command 2> errors.txt`

// Both to file
await $`command > output.txt 2>&1`

// stdin from file
await $`sort < unsorted.txt`
```

### Global Configuration

```typescript
import { $ } from 'bun'

// Set defaults for all subsequent $ calls
$.cwd('/project/root')
$.env({ NODE_ENV: 'production', ...process.env })
$.throws(false)              // Global nothrow
$.quiet(true)                // Global quiet
```

### Shell Builtins

Bun's shell supports these builtins (cross-platform):
- `cd`, `echo`, `pwd`, `which`, `rm`, `mv`, `cp`, `mkdir`, `cat`, `touch`
- `ls`, `exit`, `true`, `false`, `yes`, `seq`, `dirname`, `basename`
- `source`, `export`, `test`/`[`

### Error Handling

```typescript
import { ShellError } from 'bun'

try {
  await $`false`
} catch (err) {
  if (err instanceof ShellError) {
    err.exitCode     // number
    err.stderr       // Buffer
    err.stdout       // Buffer
    err.message      // Error message
  }
}
```

## Bun.spawn()

Lower-level process spawning for more control.

```typescript
const proc = Bun.spawn(command: string[], options?: SpawnOptions): Subprocess
```

### SpawnOptions

```typescript
interface SpawnOptions {
  cwd?: string
  env?: Record<string, string | undefined>
  stdin?: 'pipe' | 'inherit' | 'ignore' | null | BunFile | Blob | Response | ReadableStream | ArrayBuffer | Uint8Array | number
  stdout?: 'pipe' | 'inherit' | 'ignore' | null | BunFile | number
  stderr?: 'pipe' | 'inherit' | 'ignore' | null | BunFile | number
  onExit?: (proc: Subprocess, exitCode: number | null, signalCode: string | null, error: Error | undefined) => void
  windowsHide?: boolean
  windowsVerbatimArguments?: boolean
  ipc?: (message: any, subprocess: Subprocess) => void  // IPC handler
  serialization?: 'json' | 'advanced'                   // IPC serialization
}
```

### Subprocess Interface

```typescript
interface Subprocess {
  readonly pid: number
  readonly stdin: FileSink | undefined       // If stdin: 'pipe'
  readonly stdout: ReadableStream | undefined // If stdout: 'pipe'
  readonly stderr: ReadableStream | undefined // If stderr: 'pipe'
  readonly exited: Promise<number>           // Exit code promise
  readonly exitCode: number | undefined      // Available after exit
  readonly signalCode: string | undefined    // If killed by signal
  readonly killed: boolean

  kill(signal?: number | string): void       // Send signal
  ref(): void                                // Prevent process exit
  unref(): void                              // Allow process exit
  send(message: any): void                   // IPC send (if ipc option set)
  disconnect(): void                         // Close IPC channel
}
```

### Usage Examples

```typescript
// Basic: inherit all stdio
const proc = Bun.spawn(['ls', '-la'], {
  stdout: 'inherit',
  stderr: 'inherit',
})
await proc.exited

// Capture output
const proc = Bun.spawn(['echo', 'hello'], {
  stdout: 'pipe',
})
const output = await new Response(proc.stdout).text()

// Pipe input
const proc = Bun.spawn(['cat'], {
  stdin: 'pipe',
  stdout: 'pipe',
})
proc.stdin.write('hello world')
proc.stdin.end()
const result = await new Response(proc.stdout).text()

// Write stdout to file
const proc = Bun.spawn(['ls'], {
  stdout: Bun.file('listing.txt'),
})
await proc.exited

// Inherit env and add custom vars
const proc = Bun.spawn(['printenv'], {
  env: { ...process.env, CUSTOM: 'value' },
  stdout: 'pipe',
})
```

## Bun.spawnSync()

Synchronous process execution.

```typescript
const result = Bun.spawnSync(command: string[], options?: SpawnOptions): SyncSubprocess
```

### SyncSubprocess

```typescript
interface SyncSubprocess {
  readonly exitCode: number
  readonly stdout: Buffer
  readonly stderr: Buffer
  readonly success: boolean              // exitCode === 0
}
```

### Examples

```typescript
const result = Bun.spawnSync(['git', 'status'], {
  cwd: '/repo',
})

if (result.success) {
  console.log(result.stdout.toString())
} else {
  console.error(result.stderr.toString())
}
```

## IPC (Inter-Process Communication)

```typescript
// Parent process
const child = Bun.spawn(['bun', 'child.ts'], {
  ipc(message) {
    console.log('From child:', message)
  },
  serialization: 'json',  // or 'advanced' for structured clone
})

child.send({ type: 'start', data: [1, 2, 3] })

// child.ts
process.on('message', (message) => {
  console.log('From parent:', message)
  process.send({ type: 'result', value: 42 })
})
```

## Signal Handling

```typescript
process.on('SIGINT', () => {
  console.log('Interrupted')
  process.exit(0)
})

process.on('SIGTERM', () => {
  console.log('Terminated')
  cleanup()
  process.exit(0)
})

// Kill a subprocess
proc.kill()              // SIGTERM
proc.kill('SIGKILL')     // Force kill
proc.kill(9)             // Signal number
```
