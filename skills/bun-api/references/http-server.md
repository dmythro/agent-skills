# HTTP Server Reference

> Full API: `node_modules/bun-types/docs/runtime/http/server.mdx`,
> `http/routing.mdx`, `http/websockets.mdx`, `http/tls.mdx`, `http/error-handling.mdx`.
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
    '/*': () => new Response('Not Found', { status: 404 }),
  },

  fetch(req) {
    return new Response('Unmatched', { status: 404 })
  },
})
```

**Precedence**: exact (`/users/all`) > parameter (`/users/:id`) > wildcard (`/users/*`) >
global catch-all (`/*`). Route parameters are percent-decoded automatically.

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
(`304`), and `Range` (`206`). A directory without a trailing slash gets a `301`; with one,
`index.html` is served. Missing files return `404`.

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

### Conditional Requests (v1.4+)

Static routes, directory routes, and `Bun.file()` bodies also handle conditional requests:
`If-None-Match` / `If-Modified-Since` return `304`, and `If-Match` / `If-Unmodified-Since`
return `412` when the precondition fails (both were ignored before 1.4).

## Backpressure (v1.4+)

`ReadableStream`, `WritableStream`, and `TransformStream` are native in 1.4, and `Bun.serve`
pauses a streaming request or response body when the connection cannot accept more data --
a slow client holds at most one buffer's worth of server memory. The same applies to
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

Manual throttling written for 1.3 can usually be removed. Buffered consumers (`.text()`,
`.json()`, `.arrayBuffer()`) opt out and still receive the whole body at once.

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

    // Options
    maxPayloadLength: 16 * 1024 * 1024,  // 16 MB (default)
    idleTimeout: 120,                     // seconds (default: 120)
    backpressureLimit: 1024 * 1024,       // 1 MB
    perMessageDeflate: true,              // Compression
  },
})
```

### WebSocket Methods

```typescript
interface ServerWebSocket<T> {
  data: T                               // Custom data from upgrade
  readyState: number                    // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED
  remoteAddress: string

  send(data: string | Uint8Array | ArrayBuffer): void
  close(code?: number, reason?: string): void

  // Pub/Sub
  subscribe(topic: string): void
  unsubscribe(topic: string): void
  publish(topic: string, data: string | Uint8Array): void
  isSubscribed(topic: string): boolean

  // Backpressure
  cork(callback: () => void): void      // Batch multiple sends
}
```

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
