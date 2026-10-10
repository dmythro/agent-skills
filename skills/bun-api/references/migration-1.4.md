# Bun 1.3 to 1.4: Runtime Behavior Changes

**What changed under existing code.** Bun's shipped docs describe the current state only --
this file is for editing or upgrading a codebase written against 1.3.x, plus the 1.4.1, 1.4.2,
and 1.4.3 changes at the end. CLI and package manager changes live in the `bun-cli` skill's
`references/migration-1.4.md`.

Check `bun --version` before applying any of this. On 1.3.x the pre-1.4 behavior still holds.

## Most Likely to Bite

1. **`Bun.cron` now uses local time.** `cron.parse()` and in-process `cron(schedule, handler)`
   read schedules in the process time zone; they used UTC before. Add `{ tz: "UTC" }` as the
   final argument to keep 1.3 timings.
2. **Bun invoked as `node` no longer loads `.env`.** Under `bun --bun`, `bunx --bun`, or a
   `node` symlink, `.env`/`.env.local`/`.env.{development,production,test}` are skipped, matching
   Node. `bun file.js` still loads them. Pass `--env-file` to restore.
3. **Duplicate headers are joined with `", "`.** On `fetch()` responses and `Bun.serve` requests,
   per the Fetch spec. Only the last value was kept before. Empty header values now read `""`
   instead of `null`. `getSetCookie()` is unaffected.
4. **`Temporal` is defined by default.** `Bun.deepEquals()`, `toEqual()`, `toStrictEqual()`, and
   `util.isDeepStrictEqual()` compare Temporal objects **by value**; any two instances of the
   same class compared equal before. Disable with `BUN_JSC_useTemporal=0`.
5. **Node 26 is the compatibility target.** `process.versions.modules` is `147` -- prebuilt
   native addons need a `147` build. `res.writeHeader()` is removed (use `res.writeHead()`).
   In paused mode, `readable.read()` with no size returns one chunk, not the whole buffer.

## fetch() and HTTP Client

- **Network errors reject with `TypeError`**, not `Error`. `.code` (e.g. `ECONNRESET`) is still
  set. After a failed body read, `bodyUsed` is `true` and a second read rejects with
  `ERR_BODY_ALREADY_USED` -- issue a fresh `fetch()` to retry.
- **`Request#clone()` / `Response#clone()` throw** once the body is read or the stream is
  locked (`TypeError: Body is disturbed or locked`). Includes the request handed to `Bun.serve`
  route handlers. Clone **before** reading.
- **`checkServerIdentity` runs before the request is sent**, and again on each redirect hop. If
  it returns an `Error`, nothing is written. Certificate pinning across redirects must accept
  every hop, or use `redirect: "manual"`.
- **`redirect: "error"` rejects only on 301/302/303/307/308.** Other `3xx` (including `304`)
  now resolve.
- **Latin-1 header values are sent byte-for-byte**, not UTF-8 encoded.
- **The idle timeout is one deadline for the whole response header block.** A server that
  trickles header bytes now times out; each byte used to reset the timer.
- `Connection`, `Transfer-Encoding`, `Content-Encoding`, and `Upgrade` are parsed as token lists.

## Bun.serve

- **`server.publish()` / `ws.publish()` / `publishText()` / `publishBinary()` return `0` or
  `-1`.** `0` = dropped or no subscribers, `-1` = a subscriber has backpressure, byte count
  otherwise. They previously returned the byte count whenever a subscriber existed, even when
  the data was discarded.
- **`server.stop()` closes idle connections and waits for in-flight requests.**
- **`Bun.serve({ port })` throws `RangeError`** for non-integer, negative, or out-of-range
  ports. `port: 65536` silently clamped to 65535 before, and `port: -1` bound a random port.
- **A returned `Response` with a status outside 100-999** (e.g. `Response.error()`) is treated
  as a thrown error: it goes to `error()` and answers `500`.
- **Per-method route objects answer `HEAD` with the `GET` handler** when no `HEAD` key is set.
- **`server.upgrade()` returns `false`** unless the request carries `Upgrade: websocket` and a
  well-formed `Sec-WebSocket-Key`; `426` when `Sec-WebSocket-Version` is not `13`.
- **`ws.subscribe()` / `unsubscribe()` return `false`** on a closed socket.
- **`ws.send()`/`publish()` of an in-memory `Blob` sends its bytes** as a binary frame; it sent
  the text `[object Blob]` before. A `Bun.file()` blob throws -- read it first.
- **HTML routes no longer serve sourcemaps in production.** Set `[serve.static] sourcemap` in
  `bunfig.toml` to choose explicitly.
- **`Bun.serve({ inspector })` is removed** and silently ignored. Use `bun --inspect`.
- Static and file routes evaluate `If-Match` / `If-Unmodified-Since` and answer `412`.
- Malformed `Content-Length` / `Transfer-Encoding` / chunked framing gets `400`.

## TLS and Sockets

- **`Bun.connect({ tls })`, `socket.upgradeTLS()`, and `Bun.listen({ requestCert: true })`
  default to `rejectUnauthorized: true`.** A self-signed dev server without a `ca` no longer
  throws -- instead `handshake` runs with `socket.authorized === false`, writes return `-1`,
  and the socket closes without delivering data. Pass the CA, or `rejectUnauthorized: false`.
- **`tls.connect({ host, port })` uses `host` as the default `servername`** (v1.3.13+).
  Connecting by IP or to `localhost` fails with `ERR_TLS_CERT_ALTNAME_INVALID` when the
  certificate names something else. Applies through drivers like `pg` and `ioredis` too.
- **`tls.createServer({ requestCert: true })` rejects unverified client certificates.**
  `requestCert` must be literally `true`; only a literal `rejectUnauthorized: false` disables
  verification (`null` no longer does). `tls.Server` ignores `NODE_TLS_REJECT_UNAUTHORIZED`.
- **`net.Server` / `tls.Server` no longer auto-resume accepted sockets** -- bytes arriving
  before a `'data'` listener are buffered, as in Node.
- **`Bun.Socket#setKeepAlive(true, delay)` treats `delay` as milliseconds.** It was used raw as
  seconds, so `4000` meant 4000 seconds. Values under 1000 now round to 0 and leave
  `TCP_KEEPIDLE` unchanged. Pass `60_000`, not `60`.
- `X509Certificate#serialNumber` and `.modulus` return **uppercase** hex.
- `dgram.Socket` throws synchronously (`ERR_SOCKET_ALREADY_BOUND`,
  `ERR_SOCKET_DGRAM_NOT_RUNNING`) instead of emitting `error`.

## Databases

- **`Bun.sql` decodes MySQL `DATETIME` and `TIMESTAMP` as UTC.** Values came back shifted by
  the host's UTC offset before. Remove any offset correction. Postgres `timestamp` read via
  `.simple()` is UTC too; `timestamptz` is unaffected.
- **`Bun.sql` parses MariaDB 10.5+ `JSON` columns** and JSON function results into objects.
  Remove the `JSON.parse()`.
- **Postgres `infinity` / `-infinity` dates/timestamps** decode to the numbers `Infinity` /
  `-Infinity`, not an invalid `Date`. Check before calling `Date` methods.
- **`PGSSLMODE` is honored** from the environment (a `?sslmode=` in the URL still wins).
  `PGSSLMODE=require` against a non-TLS server now fails instead of connecting in plaintext.
- **`connectionTimeout` bounds the whole handshake** instead of restarting per packet.
- **`bun:sqlite` `db.close()` finalizes every `db.query()` statement**, not just cached ones.
  It threw `database is locked` before. A finalized statement throws when used;
  `db.close(true)` also finalizes `db.prepare()` statements.
- **`new Bun.RedisClient(url)` throws** when the URL path is not a database index
  (`redis://host/notadb`); it connected to database `0` before.
- **`rediss://` verifies the TLS hostname** and rejects the first command with
  `ERR_TLS_CERT_ALTNAME_INVALID` on mismatch. Connect by the certificate's name, or pass
  `tls: { rejectUnauthorized: false }`.

## Shell, Processes, FFI

- **`Bun.$` globs only patterns written in the template.** Glob characters arriving through
  `${...}`, a shell variable, command substitution, or quotes are literal. `` $`echo ${"**/"}*` ``
  now fails with `no matches found` -- write the pattern inline. `?`, `[...]`, and a leading
  `!` are literal everywhere.
- **`Bun.$` fails with `ambiguous redirect`** when a redirect target expands to more than one
  word; the words were joined into one path before.
- **`Bun.spawn()` / `Bun.spawnSync()` throw** on a NUL byte in `argv0` or `cwd`
  (`ERR_INVALID_ARG_VALUE`), on `timeout: NaN` (`ERR_OUT_OF_RANGE`), on `killSignal: 0`
  (`ERR_UNKNOWN_SIGNAL`), and on an already-aborted `signal` (`AbortError`, no process created).
- **`bun:ffi` is engine-native.** `returns: "cstring"` and `cstring` callback arguments are
  **plain strings** (`NULL` is `null`); `new CString(ptr)` returns a string with no `.ptr`,
  `.byteLength`, or `.arrayBuffer` -- keep the original pointer if you need to free it.
  `napi_env`/`napi_value` argument types throw outside `cc()`, and entry points throw when the
  JIT is disabled. New `buffer_length` argument type passes a TypedArray's length with its pointer.
- **`Bun.Terminal#write()` returns the full input length** because the whole input is buffered;
  re-sending the unflushed remainder used to duplicate input.

## Files, Modules, Parsers

- **`fs.rmdir` rejects `{ recursive: true }`** with `ERR_INVALID_ARG_VALUE`. Use
  `fs.rm(path, { recursive: true, force: true })`.
- **Exceptions thrown inside `node:fs`, `node:dns`, and `crypto.pbkdf2` callbacks** are now
  `uncaughtException`, not `unhandledRejection`. Move the handler.
- **`dns.lookup()` uses the system resolver on Linux** (`getaddrinfo`), not c-ares.
  `dns.setServers()` no longer affects it. `dns.resolve*()` and `Bun.dns.lookup()` still use
  c-ares; pass `{ backend: "c-ares" }` to `Bun.dns.lookup()` for the old path.
- **`Bun.mmap(path, { offset })` starts the view at `offset`.** It rounded down to a page
  boundary before -- remove any `offset % pageSize` compensation.
- **`import "."` and `import ".."` resolve as directories** (index file / `package.json` `main`),
  matching Node. `"."` inside `lib/run.ts` loaded `lib.ts` before; it now loads `lib/index.ts`.
- **`.css` imports at runtime export `{}`**, not the file's absolute path.
- **`.xml` imports return the parsed document**, not the path. Use `--loader .xml:file` to revert.
- **`Bun.TOML.parse()` and `bunfig.toml` throw `SyntaxError`** on unquoted string values,
  missing newlines between pairs, and integers past `Number.MAX_SAFE_INTEGER`. An unquoted
  `bunfig.toml` value now fails at startup -- quote it.
- **`Bun.JSONC.parse()` throws `SyntaxError`** on invalid input and on `""` (returned `{}` before).
- **`Bun.YAML` follows YAML 1.2**: `yes`/`no`/`on`/`off`/`y`/`Y` parse as strings. An `on:` key
  in a GitHub Actions workflow is the string `"on"`. `YAML.parse()` throws `SyntaxError` on a
  NUL byte -- pad buffers with newlines instead of zeros.

## TypeScript and JSX

- **`"jsx": "react-jsx"` emits `jsx`/`jsxs` from `<pkg>/jsx-runtime`**, the production runtime.
  Both used to import `jsxDEV` unless `NODE_ENV=production`. Set `"jsx": "react-jsxdev"` to
  keep the development runtime.
- **`useDefineForClassFields: false` is honored.** Instance field initializers move into the
  constructor after parameter-property assignments, and declaration-only fields are dropped.
  The option was previously ignored.

## Test Runner (`bun:test`)

- **`jest.resetAllMocks()` / `vi.resetAllMocks()` drop mock implementations**, not just call
  history -- matching Jest. A `jest.fn(() => 42)` returns `undefined` afterwards. Use
  `clearAllMocks()` if you only wanted the history cleared.
- **`expect().toContain()` compares with `===`**, not `Object.is`. `expect([-0]).toContain(0)`
  passes and `expect([NaN]).toContain(NaN)` fails. `toBe()` still uses `Object.is`;
  `toContainEqual()` still uses deep equality.

## Smaller Behavior Changes

- `Bun.password.hash()` with argon2 requires `memoryCost >= 8`. Existing 1.3 hashes still verify.
- `Bun.randomUUIDv7()` throws `RangeError` for timestamps at or above `2**48`, `NaN`, an invalid
  `Date`, or a `Date` before 1970. These were previously truncated or encoded as `0`.
- `Bun.color()` output changed for `"ansi-16"`, `"hsl"`, `"lab"`, and near-black `"ansi-256"`.
  A 24-bit number such as `0xff0000` is now opaque; it had alpha `0` before.
- `Bun.Cookie` serializes `Expires` like `Date#toUTCString()` -- the weekday was off by one and
  the zone was `-0000` instead of `GMT`.
- `Bun.FileSystemRouter.match()` returns `null` for a non-empty path not starting with `/`.
- `Bun.udpSocket({ connect: { port } })` throws for a port outside 1-65535.
- `S3Client.list()` entries expose `checksumAlgorithm`; the misspelled `checksumAlgorithme` is
  non-enumerable and no longer appears in `Object.keys()` or `JSON.stringify()`.
- `structuredClone()` and worker `transferList` throw `TypeError` for a non-object transfer entry.
- `bun:sqlite` keeps a column aliased `AS ""`; `columnNames` throws after `finalize()`.
- `new WebSocket(url, { proxy })` throws `SyntaxError` at construction for a non-HTTP(S) proxy
  scheme. `WebSocket#close()`, `ping()`, `pong()` validate arguments; the handshake fails if a
  requested subprotocol is not negotiated; `close()` no longer fires `close` before returning.
- ESM imports of `"bun"` and builtins evaluate exports lazily -- `Bun.redis` with an invalid
  `REDIS_URL` now throws at the binding that uses it, not at module load.
- Odd-length hex to `Bun.CryptoHasher#update()`, a primitive `options` to `TextDecoder#decode()`,
  `NaN`/`undefined` seconds to `RedisClient#expire()`, out-of-range `Bun.password` cost values,
  and `Bun.openInEditor()` with no editor found all throw instead of being silently accepted.

## 1.4.1

Behavior changes on top of 1.4.0. A 1.4.0 project sees these on `bun upgrade`.

- **`fetch()` verifies TLS against the URL hostname**, not a custom `Host` header, matching
  Node's fetch and curl. A request that used `headers: { Host }` to pick the certificate name
  now fails verification -- pass `tls: { serverName: "internal.example" }` instead (the typed
  key; lowercase `servername` is accepted too). The `Host` header is still sent.
- **`localhost` and `*.localhost` resolve to `::1` / `127.0.0.1` without the system resolver**
  in `fetch()`, `WebSocket`, `Bun.connect()`, and `bun install`. An `/etc/hosts` entry pointing
  `localhost` elsewhere no longer applies; `http://app.localhost:3000` works on every OS and
  inside Docker.
- **`fetch(url, { unix })` reuses connections.** Sequential requests share one socket where each
  opened its own before. A relative `unix` path resolves against the current `cwd` at call
  time, not the startup directory.
- **`ws` servers in `binaryType: "arraybuffer"` mode emit `ArrayBuffer`**, not `Uint8Array`,
  matching npm `ws`. `ServerWebSocket` and `ws` also accept `binaryType: "blob"`.
- **`AsyncLocalStorage.enterWith()` inside a timer or I/O callback** no longer leaks the store
  into unrelated callbacks, matching Node. Code that read context through that leak loses it.
- **A failed `import()` is retried** on the next call instead of rejecting with the cached
  error. `Promise.try` follows the updated spec. `WebAssembly.Module.imports()` / `exports()`
  descriptors drop the non-standard `type` field.
- **`util.inspect()` stops at `maxArrayLength`** instead of visiting every element.
- **`process.exit()` skips N-API addon finalizers and cleanup hooks**, as Node does.
- **Out-of-memory in `Blob.text()`, `Response.text()`, `TextDecoder.decode()`,
  `buffer.toString()`, and `StringDecoder`** throws `ERR_MEMORY_ALLOCATION_FAILED`; it was
  `ERR_STRING_TOO_LONG` or a crash.
- **`--no-ffi-cc`** makes `cc()` from `bun:ffi` throw `ERR_FFI_CC_DISABLED`, and `--no-addons`
  now disables `cc()` too. Workers inherit both flags.

### 1.4.0 regressions fixed in 1.4.1

Upgrade instead of working around these on a 1.4.0 runtime:

- `fetch()` with a `timeout` of 4 seconds or less aborted early with `TimeoutError`.
- `tls.checkServerIdentity` opened a new TLS connection per request.
- A streamed response lost the end of its body with `ECONNRESET` when the server closed
  during a client upload.
- `dns.lookup()` on macOS failed for VPN / split-DNS hostnames.
- `new Worker()` threw `port.on is not a function` when a library replaced
  `globalThis.MessagePort`.
- `assert.deepStrictEqual()` threw for objects with different prototypes.
- `ws` `handleUpgrade()` threw after an `await`.
- On Windows, `child_process`, `Bun.spawn`, and `Bun.which` could not find `.com` executables.

## 1.4.2

A bugfix release on top of 1.4.1; nothing new to opt into. What changes under existing code:

- **`.json()` rejects with the `JSON.parse` message.** `Response`, `Request`, `Blob`, `Bun.file()`,
  and `proc.stdout` `.json()` reject invalid JSON with a `SyntaxError` whose message is what
  `JSON.parse` gives (`JSON Parse error: Expected '}'`, `JSON Parse error: Unexpected EOF`)
  instead of the generic `Failed to parse JSON`. Code that matched on the old message must
  update; matching on `error instanceof SyntaxError` is unchanged.
- **`Bun.Image` decodes CMYK and YCCK JPEGs**, converting them to RGB. They rejected with
  `Image: decode failed` before, so a fallback path that caught that error for print-oriented
  JPEGs is no longer reached.
- **JavaScriptCore upgrade** (about 350 upstream commits): `Intl.PluralRules#select()` accepts a
  `BigInt` (was `TypeError`); `Intl.DurationFormat` with `style: "digital"` no longer prints a
  stray `:` when minutes is 0 and hidden; TypedArray constructors read the right element when a
  `valueOf` mutates the source array mid-conversion, and `TypedArray.prototype.slice` into an
  overlapping `Symbol.species` view copies in spec order; `Object.setPrototypeOf(handler, null)`
  on a `Proxy` handler that had already served a trap no longer crashes; `Date` objects are
  cheaper to allocate.

### 1.4.1 regressions fixed in 1.4.2

Upgrade instead of working around these on a 1.4.1 runtime:

- **`AsyncLocalStorage` memory leak.** A timer, immediate, or pending promise created inside
  `store.exit()` or a nested `store.run()` kept the outer store value alive for as long as it
  existed. `getStore()` returned the right value throughout, so it showed up as RSS growth on
  servers that stash per-request context in the store, not as wrong reads. Verified on 1.4.2:
  the outer value is collectable once the callback returns.
- **`worker_threads` `'online'` fired after the first `'message'`.** A `Worker` that posts on
  startup lost that message when the parent attached its listener after `await once(worker,
  "online")`; `@discordjs/ws` hung on it. The order now matches Node.

### Other 1.4.2 fixes

- A rare crash in long-running processes after a prototype that JIT-optimized code had cached
  property lookups through was garbage-collected.
- On musl (Alpine images), `Array.prototype.splice`, `shift`, or shrinking `array.length` on an
  array of objects could crash on a GC thread or hang while the GC was marking.
- On Linux, when registering a `Bun.file().writer()` `FileSink` with epoll failed (for example
  with `fs.epoll.max_user_watches` exhausted), its file descriptor was closed twice, which could
  close an unrelated descriptor that had reused the number.

## 1.4.3

New APIs -- `Bun.FetchSession`, `Bun.ModuleGraph`, `CompressionStream` levels, `If-Range` -- are
in the SKILL. What changes under existing code is below. Verified on the 1.4.3 runtime against
1.4.2 except items marked (release notes).

### Most Likely to Bite

1. **A per-request `tls.checkServerIdentity` costs a TLS connection per request.** The request
   opens its own connection and closes it after the response, so no other request's function
   vouches for it: 3 requests make 3 handshakes and 3 calls, where 1.4.2 reused one connection.
   Move certificate pinning to `new Bun.FetchSession({ tls: { checkServerIdentity } })`, which
   verifies once per connection and pools it. A value that is not a function now rejects with
   `ERR_INVALID_ARG_TYPE`.
2. **An unawaited promise that `fetch()` returns already rejected is an unhandled rejection.**
   `fetch("http://[bad")` with nothing awaiting it exited 0 on 1.4.2 and exits 1 on 1.4.3; the
   same holds for `Bun.write()`, `server.fetch()`, and `Bun.resolve()`. Await or `.catch()`
   fire-and-forget calls.
3. **The proxy environment reaches more requests.** `ALL_PROXY` is used when `HTTP_PROXY` /
   `HTTPS_PROXY` is unset (1.4.2 ignored it), and `NO_PROXY` gains wildcards, CIDR blocks, bare
   IPv6, and `host:port`. A proxy refusing `CONNECT` makes `fetch()` reject with
   `ERR_PROXY_TUNNEL` instead of resolving with the proxy's `407` as if the origin sent it.
   `proxy: false` and `respectNoProxy: false` are new; `NO_PROXY` bypassing an explicit `proxy`
   is not (1.4.2 did it too).
4. **`--disallow-code-generation-from-strings` is enforced.** Bun accepted the flag and ignored
   it; `eval()` and `new Function()` now throw `EvalError` process-wide, Workers included, so a
   deployment that carried the flag over from Node fails wherever a dependency compiles a
   string. `=strict` also refuses `node:vm`, `data:` / `blob:` imports, and plugin source text.
5. **`sql.begin()` rejects when Postgres rolled the transaction back.** A callback that caught a
   failed query's error resolved while the server rolled back everything; it now rejects with
   `ERR_POSTGRES_COMMIT_ROLLED_BACK` (release notes). See `sql-client.md` for the other
   `Bun.SQL` changes.
6. **Root certificates follow NSS 3.128.** Entrust Root Certification Authority, ePKI, Atos
   TrustedRoot 2011, and SecureSign Root CA12 are removed. A server chaining only to one of them
   fails verification -- pass `tls: { ca }` or run with `--use-system-ca` (release notes).

### fetch() and Streams

- `fetch("file://host/path")` rejects with `ERR_INVALID_FILE_URL_HOST` unless the host is empty
  or `localhost`; it read the local path.
- A `Content-Length` header that disagrees with a `ReadableStream` body rejects with
  `ERR_HTTP_CONTENT_LENGTH_MISMATCH` (an `Error`, not a `TypeError`) before sending; it failed
  with `ECONNRESET`.
- A `res.body` stream taken before `await res.text()` (or `json()`, `blob()`, ...) is locked
  afterwards, as in undici and browsers, so `getReader()` on it throws.
- `delete process.env.HTTPS_PROXY`, or reassigning a deleted proxy variable, applies to the next
  `fetch()`; it was ignored.
- A temporary DNS failure (SERVFAIL, every nameserver timing out) reports `EAI_AGAIN`, not
  `ETIMEOUT`, from `fetch()`, `Bun.connect()`, and `dns.lookup()` on macOS and Windows or with
  the `system` / `libc` backend. Retry logic keyed on `ETIMEOUT` must switch (release notes).
- `new Response(asyncIterable)` calls the iterator's `return()` when the consumer stops early,
  so generator `finally` blocks run; it kept pulling (release notes).
- Direct streams (`type: "direct"`): `controller.write()` returns a promise once
  `highWaterMark` bytes (64 KiB) wait for a JavaScript reader, `cancel(reason)` is called on
  `reader.cancel()` or a `break`, and writes after `close()` are dropped (release notes).
- Web streams throw `RangeError: Out of memory` at a size limit instead of aborting the process.

### Bun.serve

- `If-Range` is honored: a range request whose `If-Range` matches `Last-Modified` gets `206`,
  any other gets the full `200`. 1.4.2 always answered `206`.
- A second `Response` reusing a `ReadableStream` an earlier response sent errors; it answered
  `200` with an empty body (release notes).
- Chunked request bodies with bare-LF line endings are rejected, as llhttp does (release notes).

### Files, Processes, Runtime

- `await Bun.write(dest, Bun.file(src))` resolves to the byte count on macOS when `dest` exists
  or sits on another volume; it resolved `0` though the copy was complete.
- `Bun.file(path).exists()` checks the disk again; after one `false` it kept returning `false`
  once the file existed.
- `Bun.spawn`, `Bun.spawnSync`, and `child_process` report pipe errors (`EIO`, `ENOBUFS`)
  instead of returning truncated output, and throw `EMFILE` when out of descriptors instead of
  hanging or crashing (release notes).
- On macOS, `signalCode` and `kill()` use macOS signal numbers: a child killed by SIGBUS read
  `"SIGUSR1"`, and `kill("SIGUSR1")` sent SIGBUS (release notes). `signalCode` is typed
  `NodeJS.Signals | number | null` -- a nameless signal is its number.
- On macOS, `process.memoryUsage().rss` reports the physical footprint Activity Monitor shows,
  equal to `Bun.unsafe.memoryFootprint()` -- dashboards see lower numbers (the same script read
  84 MB on 1.4.2 and 47 MB on 1.4.3). `process.resourceUsage().maxRSS` reports its peak.
- `drainMicrotasks()` from `bun:jsc` drains only microtasks and `process.nextTick`; completed
  I/O and message callbacks wait for the current callback to return (release notes).
- Runtime plugins: `onResolve` runs once per `import` / `require()` as it executes (it could run
  up to three times, including on its own result); a `require()` of a namespaced path loads
  through `onLoad`; an `a -> b`, `b -> c` mapping loads `b` for `a`; and a throwing `onResolve`
  can be caught around the `require()` (release notes).
- `Error.stackTraceLimit` is read from the `Error` constructor when a stack is captured, as in
  V8; `node:vm` contexts start at 10 (release notes).
- Bun Shell's `.then()` / `.catch()` return a rejected promise when the shell fails to start
  (`.cwd()` to a missing directory); 1.4.2 threw `ENOENT` synchronously, so a `.catch()`
  handler never ran. `await` saw a rejection either way.

### Node.js APIs

- `http.Server#listen()` on a listening server throws `ERR_SERVER_ALREADY_LISTEN`; it leaked the
  first listener, which kept the process alive after `close()`.
- `child_process.spawnSync()` of a command that cannot be spawned returns `status: null`,
  `pid: 0`, and `output: null` (it returned `undefined`, `undefined`, `[null, null, null]`).
- `util.format("%s", x)` prints `toString()` for `Buffer`, `URL`, and `URLSearchParams`, as Node
  does; it printed `<Buffer 61 62>`.
- Per-encoding `Buffer` writers (`buf.utf8Write(123)`, `hexWrite()`, ...) throw
  `ERR_INVALID_ARG_TYPE` for a non-string; they wrote its string form.
- An exception thrown in a `node:net` `'data'` or server `'connection'` listener goes to
  `uncaughtException`, as in Node; it became the socket's `'error'` and closed the socket.
  `net.Socket#write()` returns `false` and sets `socket.errored` when a send fails at once
  (release notes).
- `crypto.hkdf()` / `hkdfSync()` with `keylen` 0 fail as in Node, and `node:fs` opens files with
  `O_CLOEXEC`, so children forked by native code (`node-pty`, an addon calling `system()`) no
  longer inherit them (release notes).
- `node:http2` servers rate-limit stream resets (CVE-2023-44487, CVE-2025-8671): the
  `streamResetBurst` and `streamResetRate` options take effect, as in Node; they were accepted
  and ignored (release notes).

### JavaScriptCore

- `BigInt("-")`, `BigInt("+")`, and `BigInt("0x")` throw `SyntaxError`; they returned `0n`.
- `(a?.b)(x)` throws `TypeError` when `a` is nullish; it returned `undefined` as if the
  parentheses were not there.
- `RegExp.escape()` leaves supplementary code points unescaped, a strict-mode async generator's
  `return someCall()` awaits the promise, and `` super.tag`x` `` passes the current `this`
  (release notes).
- WebAssembly wide arithmetic (`i64.add128`, `i64.mul_wide_u`) is on by default.

### Parsers, Images, SQLite

- `Bun.Image` decodes slightly damaged JPEGs (missing end marker, truncated scan data) that
  rejected with `ERR_IMAGE_DECODE_FAILED`; a fallback path catching that error is no longer
  reached.
- `bun:sqlite` binds more than 65535 parameters correctly and `Database.deserialize()` accepts
  an `ArrayBuffer` (`sqlite-and-data.md`).
- `YAML.stringify()` with an indent other than 2 drops the trailing space after keys, keeps
  empty `[]` / `{}` inline, and aligns nested sequences -- snapshot tests of its output change
  (release notes).
- `Bun.markdown`: a backslash before whitespace in a link destination is literal text, so
  `[a](te\ st)` is no longer a link (release notes).

### Types (`bun-types` 1.4.3)

- `TextEncoder#encodeInto()` requires both arguments; calls missing one type-checked and threw.
- `test("label", { retry: 1 }, fn)`, `test.todo("label")`, `YAML.parse(bytes)`, and
  `Blob#slice()` without `lib.dom` type check.
- `Bun.WebView` history methods are typed `goBack()` / `goForward()`, the names the runtime has
  always had; 1.4.2 declared `back()` / `forward()`.
- `@types/bun` 1.4.2 -- what `bun init` resolves on release day -- lacks every 1.4.3 API.
  Depend on `bun-types@1.4.3` (SKILL.md, "Read Bun's Own Docs First").

### Regressions Fixed in 1.4.3

Upgrade instead of working around these on 1.4.0 through 1.4.2 (release notes):

- **1.4.1:** after a few hundred `util.promisify()` calls, `util.promisify(setTimeout)` could
  return the promisified `setImmediate` or `setInterval`, so `await sleep(300)` resolved at once.
- **1.4.1:** after more than 64 concurrent `fetch()` calls to one keep-alive origin, the surplus
  idle connections were closed with RST, so servers logged `ECONNRESET`.
- **1.4.1:** a `wss://` WebSocket to an IP address, or through an HTTPS proxy at one, closed
  with 1006 when a TLS 1.2 server renegotiated.
- **1.4.0:** `for await (const line of rl)` over `node:readline` threw `ERR_USE_AFTER_CLOSE` and
  dropped the last line when 1,026 or more lines arrived in one chunk (`Readable.from([text])`,
  HTTP responses).
- **1.4.0:** `destroy()` in a `'data'` listener on `child.stdout` emitted one more `'data'`, so
  `exec()` overshot `maxBuffer`; a `Bun.spawn({ terminal })` child that exited with input still
  queued leaked its PTY and spun the CPU.
- **1.4.0:** on Linux, `http2.connect()` to a closed port reported `ECONNRESET` instead of
  `ECONNREFUSED`, and was swallowed without an `'error'` listener; `crypto.randomInt()` was about
  20x slower per call.
