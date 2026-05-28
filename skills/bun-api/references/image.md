# Bun.Image

Built-in image processing -- decode, transform, and re-encode images with no native dependencies. Replaces `sharp` and `jimp`. Available as `Bun.Image` (v1.3.14+). Supported formats: JPEG, PNG, WebP, GIF, BMP, HEIC, AVIF, TIFF.

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
