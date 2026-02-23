# HTTP Server Reference

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

```typescript
const server = Bun.serve({
  // Serve static files from a directory (Bun v1.1.27+)
  static: {
    '/': new Response(Bun.file('public/index.html')),
    '/style.css': new Response(Bun.file('public/style.css')),
  },

  fetch(req) {
    // Dynamic routes handled here
    return new Response('Not Found', { status: 404 })
  },
})
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
