# Shell and Process Reference

> Full API: `node_modules/bun-types/docs/runtime/shell.mdx` and
> `runtime/child-process.mdx`. 1.4 behavior changes: `migration-1.4.md`.

## Bun.$ (Shell API)

Tagged template literal for shell commands with automatic escaping.

**Changed in 1.4 -- globbing.** Only `*`, `**`, and braces written **directly in the
template** expand. Glob characters arriving through `${...}`, a shell variable, command
substitution, or quoted text are literal. `?`, `[...]`, and a leading `!` are literal
everywhere. On 1.3.x an interpolated pattern still expanded.

```typescript
await $`echo ${"**/"}*`     // 1.3: matched recursively. 1.4: "no matches found"
await $`echo **/*`          // correct on both -- pattern is in the template
```

A redirect target that expands to more than one word (`> *.txt`) now fails with
`ambiguous redirect`; the words were joined into one path before.

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

## process.execve() (v1.3.14+)

Replace the current process image in place (POSIX `execve`). The new program keeps the same PID; on success this call never returns.

```typescript
process.execve('/usr/bin/node', ['node', 'script.js'], {
  ...process.env,
  NODE_ENV: 'production',
})
// Nothing after a successful execve() runs -- Bun has been replaced by node.
```

## Bun.Terminal (v1.3.5+, improved in 1.4)

Pseudo-terminal (PTY) for driving interactive programs with a real TTY -- replaces `node-pty`.
Works on Linux, macOS, and Windows (ConPTY). Drive `bash`, `vim`, or `htop` from JavaScript.

The usual entry point is the `terminal` option on `Bun.spawn`:

```typescript
const proc = Bun.spawn(['bash'], {
  terminal: {
    cols: 80,
    rows: 24,
    data(term, data) {
      process.stdout.write(data)      // colored output, as a real TTY would produce
    },
  },
})

proc.terminal.write('echo Hello from PTY!\n')
proc.terminal.resize(120, 40)
```

Standalone construction and control:

```typescript
const term = new Bun.Terminal(options)

term.write('ls -la\n')      // send input to the PTY
term.resize(cols, rows)     // resize the terminal
term.setRawMode(true)       // toggle raw mode
term.ref(); term.unref()    // control event-loop liveness
term.close()
```

**Changed in 1.4:** `write()` returns the full input length because the whole input is
buffered. It previously returned only the bytes flushed synchronously, so re-sending the
remainder duplicated input. `drain` now fires on POSIX.

## Bun.spawn({ cgroup }) (v1.4+, Linux)

Place a child in an existing cgroup **before** it starts, so limits such as `memory.max` and
`pids.max` apply from its first instruction and anything it forks stays inside.

```typescript
import { mkdirSync, writeFileSync } from 'node:fs'

const dir = '/sys/fs/cgroup/build-jobs'
mkdirSync(dir, { recursive: true })
writeFileSync(`${dir}/memory.max`, String(2 * 1024 ** 3))

const proc = Bun.spawn({ cmd: ['make', '-j8'], cgroup: dir })
```

Takes a path or an open file descriptor. Bun only joins the cgroup -- create, configure, and
remove it yourself. A missing directory fails the spawn; a frozen cgroup is refused with
`EBUSY`. `node:child_process` forwards the option; other platforms ignore it.

**Argument validation changed in 1.4.** `Bun.spawn()` / `spawnSync()` now throw rather than
silently coercing: `ERR_INVALID_ARG_VALUE` for a NUL byte in `argv0` or `cwd`,
`ERR_OUT_OF_RANGE` for `timeout: NaN`, `ERR_UNKNOWN_SIGNAL` for `killSignal: 0`, and
`AbortError` for an already-aborted `signal` (no process is created).
