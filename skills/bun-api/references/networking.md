# Networking (TCP / UDP / WebSocket / fetch)

Low-level socket APIs and HTTP client behavior. For the HTTP server, see `http-server.md`.

> Full API: `node_modules/bun-types/docs/runtime/networking/{fetch,tcp,udp,dns}.mdx`.
> 1.4 behavior changes -- including TLS defaults that can break a working connection --
> are collected in `migration-1.4.md`.

**Changed in 1.4 -- TLS verification defaults.** `Bun.connect({ tls })`,
`socket.upgradeTLS()`, and `Bun.listen({ requestCert: true })` now default to
`rejectUnauthorized: true`, as `node:tls` and `fetch()` already did. Connecting to a dev or
staging server with a self-signed certificate and no `ca` does **not** throw -- the
`handshake` handler runs with `socket.authorized === false`, writes return `-1`, and the
socket closes without delivering data. Pass the CA in `tls`, or `rejectUnauthorized: false`.

## TCP Sockets

`Bun.listen()` (server) and `Bun.connect()` (client) provide raw TCP with an event-handler `socket` object.

```typescript
// Server
const server = Bun.listen({
  hostname: '127.0.0.1',
  port: 8080,
  socket: {
    open(socket) { socket.write('welcome\n') },
    data(socket, data) { /* data is a Buffer */ },
    close(socket) {},
    drain(socket) {},          // backpressure relieved
    error(socket, err) {},
  },
})
server.stop()                  // server.port holds the bound port

// Client
const client = await Bun.connect({
  hostname: '127.0.0.1',
  port: 8080,
  socket: {
    open(socket) { socket.write('ping') },
    data(socket, data) {},
  },
})
```

### Unix Domain Sockets

```typescript
Bun.listen({ unix: '/tmp/app.sock', socket: { /* ... */ } })
await Bun.connect({ unix: '/tmp/app.sock', socket: { /* ... */ } })
```

As of v1.3.12, closing a Unix-socket server removes the `.sock` file automatically, and binding to a path already in use throws `EADDRINUSE` instead of silently succeeding.

### TCP_DEFER_ACCEPT (Linux, v1.3.12+)

On Linux, `Bun.serve()` defers accepting a connection until the client sends its first bytes, reducing event-loop wake-ups. This is automatic -- no configuration needed.

## UDP Sockets

`Bun.udpSocket()` creates a datagram socket.

```typescript
const sock = await Bun.udpSocket({
  socket: {
    data(socket, buf, port, addr, flags) {
      // buf: Buffer; port/addr: sender; flags.truncated: datagram was truncated (v1.3.12+)
    },
    error(socket, err) {
      // ICMP errors (e.g. "port unreachable") now fire here instead of
      // silently closing the socket (v1.3.12+)
    },
  },
})

sock.send('hello', 41234, '127.0.0.1')
console.log(sock.port)
sock.close()
```

The `data` handler's fifth `flags` argument (v1.3.12+) reports whether an oversized datagram was `truncated`.

## WebSocket Client

The standard `WebSocket` global is built in (no `ws` package needed).

```typescript
const ws = new WebSocket('wss://example.com/socket')
ws.addEventListener('open', () => ws.send('hi'))
ws.addEventListener('message', (e) => console.log(e.data))
ws.addEventListener('close', () => {})
```

### Close Codes and Failures (Changed in 1.4.4)

The client follows browsers and Node, which changes what `close` and `error` report:

| Situation | 1.4.4 | 1.4.3 |
|---|---|---|
| `ws.close()` without a code | Empty Close frame; both ends report `1005` | `1000` |
| When `close` fires | After the server's Close frame (one more round trip), or `1006` if none arrives in 30 s (`BUN_CONFIG_WS_CLOSE_TIMEOUT`) | Right after `close()`, `wasClean: true` regardless |
| Connection fails (refused, DNS, TLS, bad handshake) | `error` with `event.error.code` (`ECONNREFUSED`, `ENOTFOUND`, `DEPTH_ZERO_SELF_SIGNED_CERT`, `ERR_TLS_CERT_ALTNAME_INVALID`) and the text in `event.message`, then `close` with `1006`, `reason: ""`, `wasClean: false` | `error` without `.error.code`, `close` with `1006` and the text in `reason`; other failures closed with `1002`-`1015` and an English reason |
| TCP connection lost after `open` | Only `close`, with `1006` | -- |

```typescript
const ws = new WebSocket('ws://localhost:1')
ws.onerror = (event: any) => console.error(event.error?.code, event.message)  // ECONNREFUSED ...
ws.onclose = (event) => console.log(event.code, event.wasClean)               // 1006 false
```

The rows about the 30 s timeout and the 1.4.3 failure codes come from the release notes; the rest
was checked on both versions. `ws.terminate()` closes immediately without the handshake.

Also as of v1.4.4 (release notes): `new WebSocket(url, { protocols: 'chat' })` works (it threw
`Value is not a sequence`), `ws://:token@host` sends the credentials, non-ASCII header values go out
as Latin-1 like `fetch`, `send()` of a `Bun.file()` or S3 file throws instead of sending an empty
frame, and a proxy's `407` is reported as such. With an `https://` proxy, `proxy.tls` configures the
proxy connection separately from the server's `tls`:

```typescript
const ws = new WebSocket('wss://example.com', {
  proxy: { url: 'https://proxy.internal:8443', tls: { ca: Bun.file('./proxy-ca.pem') } },
})
```

### The `ws` Package (Rewritten in 1.4.4)

`import { WebSocket, WebSocketServer } from "ws"` loads Bun's built-in implementation, which 1.4.4
rewrote as a port of npm `ws` 8.22.0 over the native client and server. Code written for `ws` on
Node now behaves the same:

```typescript
import { WebSocket, createWebSocketStream } from 'ws'

const ws = new WebSocket('wss://example.com/logs', { handshakeTimeout: 5_000, maxPayload: 1024 * 1024 })
ws.on('error', (err) => console.error(err.code))   // an Error with .code; 1.4.3 passed an ErrorEvent
ws.on('close', (code, reason) => {})               // reason is a Buffer; 1.4.3 passed a string
createWebSocketStream(ws, { encoding: 'utf8' }).pipe(process.stdout)   // threw "Not supported yet in Bun"
```

- **An `'error'` event without a listener is an uncaught exception** and ends the process, as on
  Node -- 1.4.3 swallowed it. Add an `'error'` listener to every client and server socket.
- `send()` while `CONNECTING` throws `WebSocket is not open: readyState 0 (CONNECTING)`.
- Server sockets have `pause()`, `resume()`, and `isPaused`, and are `instanceof WebSocket`.
- Release notes: `agent`, `createConnection`, and `lookup` are respected (SOCKS proxy agents work);
  `handshakeTimeout`, `followRedirects`, `origin`, `auth`, `autoPong`, and similar options take
  effect instead of being ignored; `maxPayload` is enforced, defaulting to 100 MiB (was 16 MiB);
  `WebSocketServer({ perMessageDeflate })` negotiates compression and accepts any `Duplex`
  (`http2.createSecureServer({ allowHTTP1: true })`, socket.io sticky sessions); and `ws` servers
  no longer inherit `Bun.serve`'s 120 s idle timeout, automatic pings, or 16 MiB backpressure limit.

### Over a Unix Socket (v1.3.13+)

`ws+unix://` and `wss+unix://` connect the WebSocket client over a Unix domain socket. The socket path and the request path are separated by `:`.

```typescript
const ws = new WebSocket('ws+unix:///tmp/app.sock:/realtime')
```

`perMessageDeflate: false` now correctly omits the compression extension header in the upgrade request (v1.3.14+).

### Pausing Reads (v1.4.1+)

`pause()` stops further reads from the underlying TCP socket: frames already decoded still
deliver their `message` events, nothing new is read, and the peer sees TCP backpressure.
`resume()` continues. Both return `true` on success. This is a Bun extension with no browser
equivalent.

```typescript
ws.pause()             // true; ws.isPaused === true
ws.resume()            // true
ws.bufferedAmount      // bytes still queued to send
```

## fetch()

`fetch()` is the standard global; Bun adds several transport controls.

### HTTP Version Selection (v1.3.14+, experimental)

The `protocol` option in `RequestInit` pins the HTTP version for a request. Accepted values: `'http1.1'`/`'h1'`, `'http2'`/`'h2'`, `'http3'`/`'h3'`. The HTTP/2 and HTTP/3 clients are **experimental** in this release.

```typescript
// Each works standalone, per request -- no flag required (verified on v1.4.0)
await fetch(url, { protocol: 'http2' })
await fetch(url, { protocol: 'http3' })
```

The global switches do something different: `--experimental-http2-fetch` (or
`BUN_FEATURE_FLAG_EXPERIMENTAL_HTTP2_CLIENT=1`) enables HTTP/2 globally instead of per
request, and `--experimental-http3-fetch` (or `BUN_FEATURE_FLAG_EXPERIMENTAL_HTTP3_CLIENT=1`)
enables the Alt-Svc auto-upgrade, where a response advertising HTTP/3 upgrades later
requests to that origin automatically.

`bun-types` 1.4.0 typed the option as `"http2" | "http1.1" | "h2" | "h1"`, so `'http3'`/`'h3'`
needed a cast; 1.4.1 adds them to the union (oven-sh/bun#39773).

### Request Compression (v1.4+)

`compress` compresses a buffered request body and sets `Content-Encoding` automatically.
`Content-Length` reflects the compressed size. Streaming bodies pass through unchanged.

```typescript
await fetch(url, {
  method: 'POST',
  body: largeJsonString,
  compress: 'gzip',                        // or true, 'deflate', 'br', 'zstd'
})

await fetch(url, { method: 'POST', body, compress: { encoding: 'zstd', level: 9 } })
```

### Proxies

`proxy` takes a URL string, a `URL`, `false` (v1.4.3+), or an object -- whose `headers` (v1.3.4+,
such as `Proxy-Authorization`) reach the proxy itself, for HTTPS and plain HTTP destinations
alike.

```typescript
await fetch(url, {
  proxy: { url: 'http://proxy.example.com:8080', headers: { 'Proxy-Authorization': 'Bearer t' } },
})
await fetch(url, { proxy: false })                                      // direct, ignoring the environment (v1.4.3+)
await fetch(url, { proxy: { url: 'http://proxy:8080', respectNoProxy: false } })  // proxy even NO_PROXY hosts (v1.4.3+)
```

Without a `proxy` option, Bun reads `HTTP_PROXY`, `HTTPS_PROXY`, and (v1.4.3+) `ALL_PROXY` --
upper or lower case -- on every request. `ALL_PROXY` applies to `http:` and `https:` URLs when
the scheme-specific variable is unset; a non-HTTP value such as `socks5://` is ignored. A request
with `unix` never uses the proxy environment. `delete process.env.HTTPS_PROXY` takes effect on
the next request as of 1.4.3 (it was ignored).

`NO_PROXY` (read when `no_proxy` is unset or empty) lists hosts that bypass the proxy, separated
by commas or whitespace. It also bypasses an explicit `proxy`, unless that sets
`respectNoProxy: false`.

| Entry | Matches |
|---|---|
| `*` | every host |
| `example.com`, `.example.com`, `*.example.com` | `example.com` and its subdomains |
| `127.0.0.1`, `::1`, `[::1]` | that IP address (bare IPv6 v1.4.3+) |
| `10.0.0.0/8`, `fd00::/8` | every address in the block (v1.4.3+) |
| `example.com:8080` | that host on that port only (v1.4.3+) |

A proxy that answers `CONNECT` with a non-2xx status makes `fetch()` reject with
`ERR_PROXY_TUNNEL` (v1.4.3+), carrying the proxy's `status`, `statusText`, and `headers`. Before,
`fetch()` resolved with the proxy's reply as if the origin had sent it -- code that checked
`res.status === 407` must catch instead:

```typescript
try {
  await fetch('https://example.com', { proxy: 'http://proxy.internal:8080' })
} catch (e: any) {
  if (e.code === 'ERR_PROXY_TUNNEL') console.error(e.status, e.headers.get('proxy-authenticate'))
}
```

### Fetch Sessions (v1.4.3+)

`Bun.FetchSession` holds connection settings and owns a keep-alive pool. `session.fetch` is
`fetch` bound to the session; `fetch(url, { session })` is the same through the global `fetch`.
Bun never shares a connection between two sessions, or between a session and plain `fetch()`.

```typescript
const PINNED = 'AB:CD:...'   // SHA-256 fingerprint of the expected leaf certificate
const pinLeafCert = (host: string, cert: { fingerprint256: string }) =>
  cert.fingerprint256 === PINNED ? undefined : new Error(`unexpected certificate for ${host}`)

using session = new Bun.FetchSession({
  tls: { ca: await Bun.file('corp-ca.pem').text(), checkServerIdentity: pinLeafCert },
  proxy: { url: 'http://proxy.internal:8080', respectNoProxy: false },
  keepAlive: { idleTimeout: 30, maxIdleSockets: 8 },
})
const client = new SomeClient({ fetch: session.fetch as typeof fetch })
```

| Option | Meaning |
|---|---|
| `tls` | As `fetch`'s `tls`. `checkServerIdentity` runs once per connection, and the connection is then pooled |
| `proxy` | As `fetch`'s `proxy`, `false` included. An unusable value (`8080`, an object without `url`) throws `ERR_INVALID_ARG_TYPE` at construction |
| `keepAlive` | `false` closes each connection after its response. `idleTimeout` in seconds (default 300; rounded up to the socket timer, which moves in 4 s steps up to four minutes); `maxIdleSockets` caps idle connections per kind (plain, TLS, Unix) -- without it, all sessions share one bounded pool per kind |
| `unix` | A Unix socket path for every request of the session |

- An option on the request overrides the session's; a request's `tls` replaces the session's
  `tls` as a whole.
- `session.close()` (or `using`) closes the idle connections; requests in flight finish, and the
  session stays usable.
- `session.fetch` has no `preconnect`, so cast it (`as typeof fetch`) where an option is typed
  `typeof fetch`.

### checkServerIdentity

`tls.checkServerIdentity(hostname, cert)` replaces Bun's hostname check and runs after the chain
verifies. Return `undefined` to approve; return or throw an `Error` to reject (the `fetch()`
promise rejects with it). As of v1.4.3, as in Node, any other truthy return also fails the
request -- a `Promise` or `true` gives `ERR_INVALID_RETURN_VALUE`, so an `async` function
approves nothing -- while `false` approves. `null` or `undefined` means no function; any other
non-function value rejects with `ERR_INVALID_ARG_TYPE`.

**Changed in 1.4.3 -- one connection per request.** A request that passes its own
`checkServerIdentity` opens a dedicated TLS connection and closes it after the response, so no
other request's function vouches for it: 3 requests cost 3 handshakes and 3 calls, where 1.4.2
reused one connection and called the function once. Certificate pinning on a hot path belongs on
a `Bun.FetchSession`, which verifies once per connection and pools it.

### Unix Socket Requests (v1.4.1+)

`fetch(url, { unix })` keeps the connection alive and reuses it like TCP -- three sequential
requests use one connection instead of three. A custom CA in `tls: { ca }` is honored, and a
relative `unix` path resolves against the current `cwd` at call time.

```typescript
await fetch('http://localhost/containers/json', { unix: '/var/run/docker.sock' })
```

**Changed in 1.4.1 -- TLS verification and `localhost`.** `fetch()` verifies the certificate
against the URL hostname, not a custom `Host` header (matching Node and curl). To verify
against a different name, pass `tls: { serverName: 'internal.example' }` (the typed key;
lowercase `servername` also works). `localhost` and
`*.localhost` resolve to `::1` / `127.0.0.1` in `fetch()`, `WebSocket`, and `Bun.connect()`
without a resolver query, so `http://app.localhost:3000` works on every OS and inside Docker.

### Response Streaming and Backpressure (v1.4+)

`fetch()` pauses reading from the socket when a delivered chunk has not been consumed, instead
of buffering the whole body. Buffered consumers (`.text()`, `.json()`, `.arrayBuffer()`) opt
out. `Response#textStream()` / `Request#textStream()` (v1.4+) give a `ReadableStream<string>`
decoded as UTF-8 -- multi-byte characters spanning chunk boundaries are reassembled, a leading
BOM is stripped, and invalid sequences become U+FFFD.

```typescript
for await (const chunk of (await fetch(url)).textStream()) {
  process.stdout.write(chunk)              // chunk is a string, not bytes
}
```

### Other Transport Behavior

- **HTTP/2 connection pooling** (v1.3.14+) -- concurrent fetches to the same origin share one multiplexed connection.
- **HTTPS proxy tunneling** (v1.3.12+) -- `CONNECT` tunnels are reused across sequential proxied HTTPS requests, including requests carrying custom `tls` options.
- **TLS session resumption** (v1.4+) -- a 32-entry per-origin LRU of BoringSSL client sessions lets a second cold connection resume at 1 RTT.
- **Faster first HTTPS request** (v1.4.1+) -- root certificates are embedded as DER and only the relevant ones are parsed, so the first TLS connection starts up to 3x sooner; an expired root no longer hides a valid one from the same issuer.
- **System CA** (v1.3.14+) -- run with `--use-system-ca`, or read the OS trust store via `tls.getCACertificates('system')` from `node:tls`.
- **`HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`** (v1.3.12+) are re-read at runtime, not only at startup; see "Proxies" for `ALL_PROXY` and the 1.4.3 syntax.
- **Root certificates** follow NSS 3.128 as of v1.4.3: Entrust Root Certification Authority, ePKI, Atos TrustedRoot 2011, and SecureSign Root CA12 were removed. A server chaining only to one of them fails verification -- pass `tls: { ca }` or run with `--use-system-ca`. v1.4.4 moves to NSS 3.131.1 (release notes).
- **TLS client fixes (v1.4.4, release notes):** servers with an Ed25519 certificate are reachable from `fetch()` and `tls.connect()`; a self-signed certificate not marked as a CA (`dotnet dev-certs`) is trusted through `NODE_EXTRA_CA_CERTS` or `ca`; assigning `tls.DEFAULT_CIPHERS` reaches `fetch`, `WebSocket`, `Bun.connect`, `RedisClient`, `Bun.SQL`, and `S3Client`; and a `tls` option given a view into a larger buffer (`buf.subarray(n)`) reads only the view.
- **`keepalive: false`** (or a `keepAlive: false` `Bun.FetchSession`) closes the connection even when the request carries `Connection: keep-alive`, as forwarded `req.headers` do; 1.4.3 left it pooled (release notes).
- **Header casing** (v1.3.7+) is preserved on the wire rather than lowercased.

**Changed in 1.4:** duplicate response headers are joined with `", "` (only the last value was
kept before); network errors reject with `TypeError` rather than `Error` (`.code` still set);
`clone()` throws once a body has been read; and `redirect: "error"` rejects only on
301/302/303/307/308, so `304` now resolves. See `migration-1.4.md`.

**Changed in 1.4.3:** `fetch("file://host/path")` rejects with `ERR_INVALID_FILE_URL_HOST`
unless the host is empty or `localhost`; a `Content-Length` header that disagrees with a
`ReadableStream` body rejects with `ERR_HTTP_CONTENT_LENGTH_MISMATCH` before sending (it failed
with `ECONNRESET`); and a `res.body` taken before `await res.text()` is locked afterwards, as in
undici and browsers.
