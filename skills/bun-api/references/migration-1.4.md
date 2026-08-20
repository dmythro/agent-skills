# Bun 1.3 to 1.4: Runtime Behavior Changes

**What changed under existing code.** Bun's shipped docs describe the current state only --
this file is for editing or upgrading a codebase written against 1.3.x. CLI and package
manager changes live in the `bun-cli` skill's `references/migration-1.4.md`.

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
