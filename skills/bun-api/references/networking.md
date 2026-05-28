# Networking (TCP / UDP / WebSocket / fetch)

Low-level socket APIs and HTTP client behavior. For the HTTP server, see `http-server.md`.

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

```typescript
// Select the HTTP version per request (v1.3.14+)
await fetch(url, { protocol: 'http2' })   // 'http1.1' | 'http2' | 'http3'
```

- **HTTP/2 connection pooling** (v1.3.14+) -- concurrent fetches to the same origin share one multiplexed connection.
- **HTTPS proxy tunneling** (v1.3.12+) -- `CONNECT` tunnels are reused across sequential proxied HTTPS requests.
- **System CA** -- run with `--use-system-ca` (or read `tls.getCACertificates('system')`) to trust the OS certificate store (v1.3.14+).
- **Experimental h2/h3** -- enable client HTTP/2 or HTTP/3 with `--experimental-http2-fetch` / `--experimental-http3-fetch` (v1.3.14+).
