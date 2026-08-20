# Bun.WebView

Headless browser automation built into the runtime -- navigate pages, run JavaScript, click, type, scroll, and capture screenshots without Playwright or Puppeteer. Available as `Bun.WebView` (v1.3.12+, improved in 1.4). Useful for scraping, screenshot generation, and end-to-end checks.

**Backends**: the system WebKit on macOS (nothing to install), or an installed Chrome,
Chromium, or Edge on macOS, Linux, and Windows. Clicks and scrolls are delivered as real user
input, not synthetic DOM events.

`Bun.WebView` extends `EventTarget` and returns `Blob` screenshots. With
`backend: "chrome"` it also exposes `.cdp(method, params?)` as an escape hatch for raw Chrome
DevTools Protocol commands -- call `navigate()` first, which sets up the CDP session. The
WebKit backend has no CDP.

> Full API -- input simulation, console capture, CDP events, persistent storage, lifecycle:
> `node_modules/bun-types/docs/runtime/webview.mdx`.

If Playwright is a hard requirement, note that it now runs on Bun as of 1.4
(`connectOverCDP()`, `playwright test`, `--ui`, Chromium on Windows).

## Launching

Use `await using` so the view is disposed (closed) automatically at scope exit.

```typescript
await using view = new Bun.WebView({ width: 800, height: 600 })

await view.navigate('https://bun.sh')
await view.click("a[href='/docs']")
await view.scrollTo('#install')

const title = await view.evaluate('document.title')
const shot = await view.screenshot({ format: 'jpeg', quality: 90 })
await Bun.write('page.jpg', shot)
```

Without `using`, call `view.close()` when done.

## Methods

| Method | Description |
|---|---|
| `navigate(url)` | Load a URL |
| `evaluate(expr)` | Run JavaScript in the page; returns the result |
| `screenshot(opts?)` | Capture an image; `opts`: `format` (`png`/`jpeg`/`webp`), `quality`, `encoding` |
| `click(selector)` / `click(x, y)` | Click an element or coordinates |
| `type(text)` | Type text into the focused element |
| `press(key, { modifiers })` | Press a key with optional modifiers |
| `scroll(dx, dy)` / `scrollTo(selector)` | Scroll by delta or to an element |
| `goBack()` / `goForward()` / `reload()` | History navigation |
| `resize(w, h)` | Resize the viewport |
| `cdp(method, params)` | Raw Chrome DevTools Protocol call |
| `close()` | Close the view (or use `await using`) |

## Properties & Events

```typescript
view.url        // current URL
view.title      // current document title
view.loading    // boolean: navigation in progress

view.onNavigated = (url) => { /* fired after a page loads */ }
view.onNavigationFailed = (err) => { /* fired on load failure */ }
```

## Extracting Data

```typescript
await using view = new Bun.WebView()
await view.navigate('https://example.com')

const links = await view.evaluate(`
  Array.from(document.querySelectorAll('a')).map(a => a.href)
`)
```

## Raw CDP

For capabilities the high-level API does not expose, call the Chrome DevTools Protocol directly:

```typescript
await view.cdp('Emulation.setDeviceMetricsOverride', {
  width: 390, height: 844, deviceScaleFactor: 3, mobile: true,
})
```
