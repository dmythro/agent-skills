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

### Over a Unix Socket (v1.3.13+)

`ws+unix://` and `wss+unix://` connect the WebSocket client over a Unix domain socket. The socket path and the request path are separated by `:`.

```typescript
const ws = new WebSocket('ws+unix:///tmp/app.sock:/realtime')
```

`perMessageDeflate: false` now correctly omits the compression extension header in the upgrade request (v1.3.14+).

## fetch()

`fetch()` is the standard global; Bun adds several transport controls.

### HTTP Version Selection (v1.3.14+, experimental)

The `protocol` option in `RequestInit` pins the HTTP version for a request. Accepted values: `'http1.1'`/`'h1'`, `'http2'`/`'h2'`, `'http3'`/`'h3'`. The HTTP/2 and HTTP/3 clients are **experimental** in this release.

```typescript
// HTTP/2 per request works standalone -- no flag required
await fetch(url, { protocol: 'http2' })

// HTTP/3 also needs the client enabled globally first:
//   bun --experimental-http3-fetch app.ts
//   (or BUN_FEATURE_FLAG_EXPERIMENTAL_HTTP3_CLIENT=1)
await fetch(url, { protocol: 'http3' })
```

Enable HTTP/2 globally (instead of per request) with `--experimental-http2-fetch`, or `BUN_FEATURE_FLAG_EXPERIMENTAL_HTTP2_CLIENT=1`.

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

### Proxy Headers (v1.3.4+)

`proxy` accepts an object, so custom headers (such as `Proxy-Authorization`) reach the proxy
itself, for HTTPS and plain HTTP destinations alike.

```typescript
await fetch(url, {
  proxy: { url: 'http://proxy.example.com:8080', headers: { 'Proxy-Authorization': 'Bearer t' } },
})
```

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
- **System CA** (v1.3.14+) -- run with `--use-system-ca`, or read the OS trust store via `tls.getCACertificates('system')` from `node:tls`.
- **`HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`** (v1.3.12+) are re-read at runtime, not only at startup.
- **Header casing** (v1.3.7+) is preserved on the wire rather than lowercased.

**Changed in 1.4:** duplicate response headers are joined with `", "` (only the last value was
kept before); network errors reject with `TypeError` rather than `Error` (`.code` still set);
`clone()` throws once a body has been read; and `redirect: "error"` rejects only on
301/302/303/307/308, so `304` now resolves. See `migration-1.4.md`.
