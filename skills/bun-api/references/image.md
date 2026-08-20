# Bun.Image

Built-in image processing -- decode, transform, and re-encode images with no native dependencies. Replaces `sharp` and `jimp`. Available as `Bun.Image` (v1.3.14+).

> Full API and the authoritative compatibility matrix:
> `node_modules/bun-types/docs/runtime/image.mdx`.

## Format Support Is Platform-Dependent

Do not assume parity across platforms -- an unsupported format rejects with
`ERR_IMAGE_FORMAT_UNSUPPORTED`.

| | Linux | macOS | Windows |
|---|---|---|---|
| JPEG, PNG, WebP, GIF, BMP | yes | yes | yes |
| HEIC / AVIF | not supported | ImageIO | WIC + Store codec |
| TIFF (decode) | not supported | ImageIO | WIC |
| Clipboard | returns `null` | NSPasteboard | Win32 |

Both OS-backed formats carry an extra condition, and they differ per platform:

- **Windows** needs the codec installed from the Microsoft Store -- **HEIF Image Extensions**
  for HEIC, **AV1 Video Extension** for AVIF. With those present, AVIF encode works.
- **macOS** can decode AVIF anywhere ImageIO does (macOS 13+), but AVIF **encode** needs an OS
  AV1 encoder, which means **Apple Silicon M3+**; Intel and M1/M2 Macs reject it.

JPEG, PNG, and WebP go through statically-linked codecs on every platform, so their encoded
output is byte-identical across Linux, macOS, and Windows. Formats handled by the system
backend inherit the OS's patch level.

## Clipboard (v1.4+)

```typescript
const img = Bun.Image.fromClipboard()      // null when there is no image, always null on Linux
Bun.Image.hasClipboardImage()
Bun.Image.clipboardChangeCount()           // poll this integer; call hasClipboardImage() when it moves
```

macOS has no clipboard-change notification, so polling `clipboardChangeCount()` is the
documented pattern. On Linux, shell out to `wl-paste`/`xclip` and pass the bytes to the
constructor.

## Constructing

```typescript
// From a Buffer / Uint8Array / ArrayBuffer
const img = new Bun.Image(buffer)

// From a file
const img = Bun.file('photo.jpg').image()

// From a Blob
const img = blob.image()
```

`width` and `height` accessors expose the source dimensions:

```typescript
const img = new Bun.Image(buffer)
img.width   // 1920
img.height  // 1080
```

## Transforms (chainable)

Transform methods return the `Bun.Image` so they can be chained; they are applied when an output method runs the pipeline.

```typescript
new Bun.Image(buffer)
  .resize(800, 600, { fit: 'contain', withoutEnlargement: true })
  .rotate(90)                       // 90 | 180 | 270
  .flip()                           // vertical
  .flop()                           // horizontal
  .modulate({ brightness: 1.1, saturation: 0.9 })
```

| Method | Description |
|---|---|
| `resize(w, h?, opts?)` | Resize; `opts`: `filter`, `fit`, `withoutEnlargement` |
| `rotate(deg)` | Rotate by `90`, `180`, or `270` degrees |
| `flip()` / `flop()` | Flip vertically / horizontally |
| `modulate(opts)` | Adjust `brightness` / `saturation` |

## Output Format

Set the encoder (chainable); each accepts format-specific options such as `quality`.

```typescript
.jpeg({ quality: 80 })
.png()
.webp({ quality: 80 })
.heic()
.avif({ quality: 50 })
```

## Producing Output

Terminal methods are async -- they run the transform pipeline and return the encoded result.

```typescript
const img = new Bun.Image(buffer).resize(800, 600).webp({ quality: 80 })

await img.bytes()            // Uint8Array
await img.buffer()           // ArrayBuffer
await img.blob()             // Blob
await img.toBase64()         // base64 string
await img.dataurl()          // 'data:image/webp;base64,...'
await img.write('out.webp')  // write the encoded image to a path
```

## metadata()

Read dimensions and format.

```typescript
const meta = await new Bun.Image(buffer).metadata()
// { width: 1920, height: 1080, format: 'jpeg' }
```

## placeholder()

Generate a tiny blur-up placeholder (thumbhash) as a data URL -- ideal for progressive image loading.

```typescript
const placeholder = await Bun.file('hero.jpg').image().placeholder()
// 'data:image/png;base64,...'  (a few hundred bytes)
```

## Clipboard (static helpers)

Read an image from the system clipboard (primarily macOS).

```typescript
Bun.Image.hasClipboardImage()   // boolean
const img = Bun.Image.fromClipboard()
```

## Common Pipeline

```typescript
// Web-ready thumbnail + blur placeholder from an upload
const src = Bun.file('upload.jpg')

const thumb = await src.image()
  .resize(400, 400, { fit: 'cover' })
  .webp({ quality: 82 })
  .bytes()

const blur = await src.image().placeholder()

await Bun.write('thumb.webp', thumb)
```
