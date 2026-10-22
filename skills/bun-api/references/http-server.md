# HTTP Server Reference

> Full API: `node_modules/bun-types/docs/runtime/http/server.mdx`, `runtime/http/routing.mdx`,
> `runtime/http/websockets.mdx`, `runtime/http/tls.mdx`, `runtime/http/error-handling.mdx`.
> Behavior that changed in 1.4 is collected in `migration-1.4.md`.

## Routes

`routes` is the primary routing API. Reach for it before hand-parsing `req.url` -- it gives
params, per-method dispatch, and zero-allocation static responses. `fetch` remains the
fallback for unmatched requests.

```typescript
Bun.serve({
  routes: {
    // Static Response -- optimized for zero-allocation dispatch
    '/health': new Response('OK'),
    '/old': Response.redirect('/new'),
    '/favicon.ico': Bun.file('./favicon.ico'),

    // Handler receives a BunRequest: Request + params + cookies
    '/users/:id': req => Response.json({ id: req.params.id }),
    '/orgs/:orgId/repos/:repoId': req => {
      const { orgId, repoId } = req.params        // typed from the literal
      return Response.json({ orgId, repoId })
    },

    // Per-method dispatch
    '/api/posts': {
      GET: () => Response.json(listPosts()),
      POST: async req => Response.json(await req.json()),
    },

    // Serve a directory (v1.4+)
    '/static/*': { dir: './public' },

    '/api/*': Response.json({ error: 'not found' }, { status: 404 }),
  },

  fetch(req) {
    return new Response('Unmatched', { status: 404 })   // no '/*' route above, so this runs
  },
})
```

**Precedence**: exact (`/users/all`) > parameter (`/users/:id`) > wildcard (`/users/*`) >
global catch-all (`/*`). A registered `'/*'` route catches every unmatched path, so `fetch`
never runs alongside one -- use either `'/*'` or `fetch` as the fallback, not both. Route
parameters are percent-decoded automatically.

**Per-method objects answer `HEAD` with the `GET` handler** when no `HEAD` key is set (v1.4+;
before, the request fell through to the next route or 404).

### Directory Routes (v1.4+)

```typescript
Bun.serve({
  routes: { '/static/*': { dir: './public' } },
})
```

Replaces `express.static`, `serve-static`, and `sirv`. Files stream with `sendfile`, and Bun
handles `Content-Type`, `Last-Modified`, a weak `ETag`, `If-None-Match`/`If-Modified-Since`
(`304`), `Range` (`206`), and `If-Range` (v1.4.3+, see Range Requests). A directory without a
trailing slash gets a `301`; with one, `index.html` is served. Missing files return `404`.

Paths are percent-decoded once and opened relative to `dir`; non-canonical paths (`.`, `..`,
empty segments, `%2F`) are rejected with `404`, and on Linux the open uses
`openat2(RESOLVE_IN_ROOT)` so symlinks cannot escape. Pass `statCache: false` to drop the
per-path `Last-Modified` cache (~20 KB per route).

Routing is case-sensitive but macOS/Windows filesystems are not -- keep access-controlled
content outside `dir` rather than gating it with an overlapping route.

## Bun.serve()

Built-in HTTP server with zero dependencies. Replaces Express, Fastify, or `http.createServer`.

```typescript
const server = Bun.serve({
  port: 3000,                          // Default: process.env.PORT || 3000
  hostname: '0.0.0.0',                 // Default: '0.0.0.0'

  fetch(req: Request): Response | Promise<Response> {
    const url = new URL(req.url)

    if (url.pathname === '/api/health') {
      return Response.json({ status: 'ok' })
    }

    if (url.pathname === '/api/data' && req.method === 'POST') {
      const body = await req.json()
      return Response.json({ received: body })
    }

    return new Response('Not Found', { status: 404 })
  },

  error(error: Error): Response {
    return new Response(`Internal Error: ${error.message}`, { status: 500 })
  },
})

console.log(`Listening on ${server.url}`)
```

## Server Properties and Methods

```typescript
interface Server {
  url: URL                              // e.g., http://localhost:3000
  port: number
  hostname: string
  development: boolean
  pendingRequests: number
  pendingWebSockets: number
  id: string                            // Unique server ID

  stop(closeActiveConnections?: boolean): void
  reload(options: ServeOptions): void   // Hot-swap config without restart
  requestIP(req: Request): SocketAddress | null
  upgrade(req: Request, options?: ServerWebSocketUpgradeOptions): boolean
}
```

### server.stop()

```typescript
// Graceful stop (wait for in-flight requests)
server.stop()

// Force stop (close all connections immediately)
server.stop(true)
```

### server.reload()

Hot-swap the server's fetch handler without restarting:

```typescript
server.reload({
  fetch(req) {
    return new Response('Updated handler')
  },
})
```

### server.requestIP()

Get the client's IP address:

```typescript
const server = Bun.serve({
  fetch(req) {
    const addr = server.requestIP(req)
    // { address: '127.0.0.1', family: 'IPv4', port: 54321 }
    return new Response(`Your IP: ${addr?.address}`)
  },
})
```

## Response Patterns

```typescript
// Plain text
new Response('Hello')

// JSON
Response.json({ key: 'value' })
Response.json({ error: 'not found' }, { status: 404 })

// HTML
new Response('<h1>Hello</h1>', {
  headers: { 'Content-Type': 'text/html' },
})

// Redirect
Response.redirect('https://example.com', 302)

// Stream
new Response(readableStream)

// File (efficient, uses sendfile)
new Response(Bun.file('public/index.html'))
```

## Streaming Responses

```typescript
const server = Bun.serve({
  fetch(req) {
    const stream = new ReadableStream({
      async start(controller) {
        for (let i = 0; i < 5; i++) {
          controller.enqueue(`data: event ${i}\n\n`)
          await Bun.sleep(1000)
        }
        controller.close()
      },
    })

    return new Response(stream, {
      headers: {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
      },
    })
  },
})
```

## Static File Serving

Prefer a directory route (see above) or `routes` entries. The older `static` option still
works at runtime but is no longer in Bun's public types or docs -- treat it as legacy and use
`routes` in new code.

```typescript
const server = Bun.serve({
  routes: {
    '/': new Response(Bun.file('public/index.html')),
    '/style.css': new Response(Bun.file('public/style.css')),
    '/assets/*': { dir: './public/assets' },       // v1.4+
  },

  fetch(req) {
    // Dynamic routes handled here
    return new Response('Not Found', { status: 404 })
  },
})

// Legacy equivalent (pre-1.2.3 codebases):
// Bun.serve({ static: { '/': new Response(Bun.file('public/index.html')) }, fetch })
```

Or dynamic static serving:

```typescript
const server = Bun.serve({
  async fetch(req) {
    const url = new URL(req.url)

    // Serve files from public/
    const file = Bun.file(`public${url.pathname}`)
    if (await file.exists()) {
      return new Response(file)
    }

    return new Response('Not Found', { status: 404 })
  },
})
```

## Range Requests (v1.3.13+)

File-backed responses (`new Response(Bun.file(...))`) automatically honor HTTP `Range` headers -- Bun returns `206 Partial Content` with the correct `Content-Range`, or `416 Range Not Satisfiable` for an invalid range. No manual handling is needed for video streaming or resumable downloads.

```typescript
Bun.serve({
  fetch(req) {
    // Range requests against this file are served automatically
    return new Response(Bun.file('./video.mp4'))
  },
})
```

### If-Range (v1.4.3+)

A client resuming a download sends `Range` with `If-Range` set to a validator from its first
response. Bun returns the range (`206`) only if that validator still matches, and the full body
(`200`) otherwise, so a resumed download never splices two versions. This covers `Bun.file()`
responses and `{ dir }` routes. Two validators match: the `Last-Modified` date, and a strong
`ETag` -- which a `Bun.file()` response carries only if you set one in its headers. A directory
route's `ETag` is weak and never matches. A `Range` request without `If-Range` still gets `206`.
Bun 1.4.2 ignored `If-Range` and always answered `206`.

```typescript
const first = await fetch('http://localhost:3000/big.bin')
const lastModified = first.headers.get('last-modified')!
// ...connection drops after 1000 bytes...
const rest = await fetch('http://localhost:3000/big.bin', {
  headers: { Range: 'bytes=1000-', 'If-Range': lastModified },
})
rest.status   // 206 if unchanged, 200 with the whole file if it changed
```

### Conditional Requests (v1.4+)

Static routes, directory routes, and `Bun.file()` bodies also handle conditional requests:
`If-None-Match` / `If-Modified-Since` return `304`, and `If-Match` / `If-Unmodified-Since`
return `412` when the precondition fails (both were ignored before 1.4).

## Backpressure (v1.4+)

`ReadableStream`, `WritableStream`, and `TransformStream` are native in 1.4, and `Bun.serve`
pauses a streaming request or response body when the connection cannot accept more data --
a slow client no longer forces the whole body into memory. This bounds the read side, not
total memory: stream queues, socket buffers, transforms, and any chunks your handler holds
still count. The same applies to
`fetch()` response bodies, `TransformStream` (including `CompressionStream`), `HTMLRewriter`,
`Bun.spawn`, `Bun.file(path).stream()`, and `Blob.stream()`.

```typescript
Bun.serve({
  routes: {
    '/': () => new Response(new ReadableStream({
      pull(controller) {
        // pauses automatically when the socket's send buffer fills
        controller.enqueue(new Uint8Array(65536))
      },
    })),
  },
})
```

Manual throttling written for 1.3 can usually be removed where every stage of the pipeline
honors backpressure. Buffered consumers (`.text()`,
`.json()`, `.arrayBuffer()`) opt out and still receive the whole body at once.

As of v1.4.3, a `new Response(asyncIterable)` body calls the iterator's `return()` as soon as the
consumer leaves -- a client disconnect, a `break`, a failed write -- so a generator's `finally`
block runs; it kept pulling before. A generator parked in an `await` stays there until it
settles, so pass `req.signal` to whatever it waits on. A `ReadableStream` that an earlier
response already sent cannot back a second `Response`: Bun now errors where it answered `200`
with an empty body.

## TLS / HTTPS

```typescript
const server = Bun.serve({
  port: 443,

  tls: {
    key: Bun.file('key.pem'),
    cert: Bun.file('cert.pem'),
    // Optional
    ca: Bun.file('ca.pem'),
    passphrase: 'password',
  },

  fetch(req) {
    return new Response('Secure!')
  },
})
```

### Client Certificates (mTLS) -- Changed in 1.4.4

A server asks clients for a certificate only with `requestCert: true`; `ca` names the CAs those
certificates are verified against. On 1.4.3, setting `ca` alone already made the server request a
client certificate -- an mTLS setup that relied on that must add `requestCert: true` on 1.4.4
(verified by watching the handshake: 1.4.3 sent a `CertificateRequest` with only `ca` set, 1.4.4
does not).

```typescript
Bun.serve({
  tls: {
    key: Bun.file('./key.pem'),
    cert: Bun.file('./cert.pem'),
    ca: Bun.file('./client-ca.pem'),      // CA that issues the client certificates
    requestCert: true,                    // required for mTLS
    crl: Bun.file('./client-ca.crl.pem'), // optional: refuse revoked client certificates
  },
  fetch: () => new Response('hi'),
})
```

With `requestCert: true`, clients without a certificate that `ca` verifies are refused unless
`rejectUnauthorized: false`; `NODE_TLS_REJECT_UNAUTHORIZED` does not change a server. The same
applies to `Bun.listen` and `node:tls` servers. A `tls` array with an RSA and an ECDSA pair now
serves each client one it supports (1.4.3 served only the last pair; release notes).

Multiple TLS certificates (SNI):

```typescript
Bun.serve({
  tls: [
    { serverName: 'a.example.com', key: Bun.file('a.key'), cert: Bun.file('a.cert') },
    { serverName: 'b.example.com', key: Bun.file('b.key'), cert: Bun.file('b.cert') },
  ],
  fetch(req) { return new Response('Hello') },
})
```

### HTTP/2 (v1.4.1+, experimental)

`http2: true` serves HTTP/2 on the same port, with the same `routes` and `fetch` handler, as
HTTP/1.1:

```typescript
Bun.serve({
  tls: { key: Bun.file('key.pem'), cert: Bun.file('cert.pem') },
  http2: true,
  http1: false,                        // optional: refuse HTTP/1.x clients
  fetch(req) { return new Response('Hello over HTTP/2') },
})
```

- With `tls`, ALPN picks the protocol per connection: clients offering `h2` (browsers, curl,
  `node:http2`) get HTTP/2, everyone else HTTP/1.1.
- Without `tls`, a connection that opens with the HTTP/2 preface (`curl --http2-prior-knowledge`,
  `http2.connect("http://...")`) gets HTTP/2; other connections get HTTP/1.1 (verified on v1.4.1).
- `http1: false` fails the handshake for TLS clients that offer ALPN without `h2`; clients that
  send no ALPN, and cleartext connections without the preface, receive `505`. It requires
  `http2: true` or `http3: true`, and it disables WebSockets, since `server.upgrade()` is
  HTTP/1.1-only.
- Not supported over HTTP/2: WebSockets, server push, and response trailers (so no gRPC).

### HTTP/3 (QUIC) (v1.3.14+)

Serve HTTP/3 over QUIC (UDP/443) alongside HTTP/1.1 and HTTP/2 by setting `http3: true` (requires TLS):

```typescript
Bun.serve({
  port: 443,
  tls: { key: Bun.file('key.pem'), cert: Bun.file('cert.pem') },
  http3: true,
  fetch(req) { return new Response('Served over HTTP/3') },
})
```

## WebSocket Upgrade

```typescript
const server = Bun.serve({
  fetch(req) {
    const url = new URL(req.url)

    if (url.pathname === '/ws') {
      const upgraded = server.upgrade(req, {
        data: { userId: url.searchParams.get('id') },
      })
      if (upgraded) return undefined  // Bun handles the response
      return new Response('Upgrade failed', { status: 400 })
    }

    return new Response('Hello')
  },

  websocket: {
    open(ws) {
      console.log('Connected:', ws.data.userId)
      ws.subscribe('chat')          // Join a pub/sub topic
    },

    message(ws, message) {
      // Broadcast to all subscribers
      ws.publish('chat', `${ws.data.userId}: ${message}`)
    },

    close(ws, code, reason) {
      ws.unsubscribe('chat')
    },

    drain(ws) {
      // Called when backpressure is relieved
    },

    error(error, ws) {
      // A handler above threw; `ws` is that socket (v1.4.4+; 1.4.3 passed only the error)
      ws.close(1011)
    },
    // ping(ws, data) {}, pong(ws, data) {} -- every handler is optional, `message` included

    // Options
    maxPayloadLength: 16 * 1024 * 1024,  // 16 MB (default)
    idleTimeout: 120,                     // seconds (default: 120); 0 = never (fixed in 1.4.4)
    backpressureLimit: 1024 * 1024,       // 1 MB
    perMessageDeflate: true,              // Compression
    binaryType: 'nodebuffer',             // 'nodebuffer' (default) | 'arraybuffer' | 'uint8array' | 'blob' (v1.4.1+)
  },
})
```

**Check `Origin` before `server.upgrade()`.** The same-origin policy does not apply to WebSocket
handshakes, so a page on another site can open a socket that carries your user's cookies
(cross-site WebSocket hijacking). Reject origins outside an allowlist with `403` first.

### Pausing a Fast Client (v1.4.4+)

```typescript
import { createWriteStream } from 'node:fs'

const sink = createWriteStream('./uploads.log')   // write() returns false when its buffer is full

Bun.serve({
  fetch(req, server) {
    if (server.upgrade(req)) return
    return new Response('Upgrade required', { status: 426 })
  },
  websocket: {
    message(ws, message) {
      if (!sink.write(message)) {
        ws.pause()                          // stop reading: the client sees TCP backpressure
        sink.once('drain', () => ws.resume())
      }
    },
  },
})
```

`ws.isPaused` reports the state, and the idle timeout is suspended while paused. Messages already
read from the socket are still delivered.

### Close Codes (Changed in 1.4.4)

| Event | 1.4.4 | 1.4.3 |
|---|---|---|
| `ws.close()` without a code (either side) | `1005` on both ends | `1000` |
| `server.stop(true)` | `1001 Server closed` | `1006 Connection ended` |
| Protocol error, invalid UTF-8, message over `maxPayloadLength` | Close frame with `1002` / `1007` / `1009` and a reason | `1006` (release notes) |

### Compression

`perMessageDeflate: true` negotiates compression, but `ws.send()` and `ws.publish()` compress
only when you pass `true` as the last argument, while `server.publish()` compresses unless you
pass `false` -- short messages grow when compressed. An object configures each direction:

```typescript
perMessageDeflate: { compress: '16KB', decompress: 'shared' }
```

`"shared"` / `true` uses one compressor for all sockets; `"3KB"` to `"256KB"` (and `"dedicated"`,
the same as `"256KB"`) keep a per-socket compressor of that size. As of v1.4.4 those sizes are
honored -- 1.4.3 treated them all as `"3KB"` -- so per-connection memory grows (`"dedicated"`:
149 KB to 282 KB), and `{ compress: false }` really stops compressing (release notes).

### WebSocket Methods

```typescript
interface ServerWebSocket<T> {
  data: T                               // Custom data from upgrade
  readyState: number                    // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED
  remoteAddress: string

  send(data: string | BufferSource | Blob, compress?: boolean): number   // bytes, 0 dropped, -1 backpressure
  close(code?: number, reason?: string): void
  terminate(): void                     // Close without the closing handshake

  // Pub/Sub
  subscribe(topic: string): boolean     // false on a closed socket
  unsubscribe(topic: string): boolean
  publish(topic: string, data: string | BufferSource | Blob, compress?: boolean): number
  isSubscribed(topic: string): boolean

  // Backpressure
  cork(callback: () => void): void      // Batch multiple sends
  pause(): void                         // Stop reading from the client (v1.4.4+)
  resume(): void                        // Read again (v1.4.4+)
  readonly isPaused: boolean            // v1.4.4+
}
```

`send()` and `publish()` return the bytes sent, `0` when the message was dropped (closed socket,
or more than `backpressureLimit` already buffered -- and for an empty payload), or `-1` under
backpressure, after which `drain` runs once the buffer empties. A Close frame and messages that
`send()` already accepted are no longer dropped by `ws.close()` over the backpressure limit,
`terminate()`, or `server.stop(true)` (v1.4.4, release notes).

## Error Handler

```typescript
Bun.serve({
  fetch(req) {
    throw new Error('something broke')
  },

  error(error) {
    // Called when fetch() throws
    console.error(error)
    return new Response(`Error: ${error.message}`, { status: 500 })
  },
})
```

If `error()` itself throws or is not provided, Bun returns a default 500 response.

## Development Mode

```typescript
Bun.serve({
  development: true,  // Default: true when NODE_ENV !== 'production'
  // Shows detailed error pages in browser
  // Disables some production optimizations
})
```

## Unix Socket

```typescript
Bun.serve({
  unix: '/tmp/my-app.sock',
  fetch(req) {
    return new Response('Hello from socket')
  },
})
```
