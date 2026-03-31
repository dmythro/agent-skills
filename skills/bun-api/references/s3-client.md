# S3 Client Reference

## Bun.s3

Built-in S3 client with Web standard Blob API compatibility. Zero dependencies, works with any S3-compatible service (AWS S3, Cloudflare R2, MinIO, DigitalOcean Spaces, etc.).

### Credentials

Bun reads S3 credentials from environment variables automatically:

```bash
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
S3_BUCKET=my-bucket
# Or S3_ENDPOINT for S3-compatible services
S3_ENDPOINT=https://s3.us-east-1.amazonaws.com
```

### Basic Operations

```typescript
import { s3, write, S3Client } from "bun"

// Create a lazy reference to an S3 file (no network request yet)
const file = s3.file("path/to/file.json")

// Read from S3
const text = await file.text()           // string
const json = await file.json()           // parsed JSON
const bytes = await file.bytes()         // Uint8Array
const stream = file.stream()             // ReadableStream
const arrayBuf = await file.arrayBuffer() // ArrayBuffer
const blob = await file.blob()           // Blob

// File metadata
file.size                                // number
file.type                                // MIME type

// Check existence
await file.exists()                      // boolean

// Upload to S3
await write(file, JSON.stringify({ name: "Alice" }))
await write(s3.file("output.txt"), "Hello, World!")
await write(s3.file("binary.dat"), new Uint8Array([1, 2, 3]))

// Delete from S3
await file.delete()
```

### Custom S3Client

```typescript
import { S3Client } from "bun"

const client = new S3Client({
  accessKeyId: "...",
  secretAccessKey: "...",
  bucket: "my-bucket",
  region: "us-east-1",
  endpoint: "https://s3.us-east-1.amazonaws.com",
  // Optional
  sessionToken: "...",
})

const file = client.file("data.json")
const data = await file.json()
```

### Presigned URLs

Generate temporary URLs for direct client access without exposing credentials. Synchronous -- no network request needed.

```typescript
import { s3 } from "bun"

// Basic presigned GET URL (default: 24 hour expiry)
const downloadUrl = s3.presign("my-file.txt")

// Presigned upload URL
const uploadUrl = s3.presign("uploads/photo.jpg", {
  method: "PUT",
  expiresIn: 3600,              // seconds (1 hour)
  type: "image/jpeg",           // content type
  acl: "public-read",           // access control
})

// Force download with specific filename
const url = s3.presign("report.pdf", {
  expiresIn: 3600,
  contentDisposition: 'attachment; filename="quarterly-report.pdf"',
})

// From file reference
const file = s3.file("data.json")
const presigned = file.presign({ expiresIn: 3600 })
```

### Presign Options

| Option | Type | Description |
|---|---|---|
| `expiresIn` | `number` | Seconds until URL expires (default: 86400 / 24h) |
| `method` | `string` | HTTP method: `"GET"`, `"PUT"` |
| `type` | `string` | Content-Type for the presigned URL |
| `acl` | `string` | Access control: `"public-read"`, `"private"`, etc. |
| `contentDisposition` | `string` | Content-Disposition header value |

### Using with Bun.serve

S3File objects work directly with Response (they implement the Blob interface):

```typescript
import { s3 } from "bun"

const server = Bun.serve({
  async fetch(req) {
    const url = new URL(req.url)

    // Serve file directly from S3
    if (url.pathname.startsWith("/files/")) {
      const key = url.pathname.slice(7)
      const file = s3.file(key)
      if (await file.exists()) {
        return new Response(file)
      }
      return new Response("Not Found", { status: 404 })
    }

    // Return presigned URL for client-side download
    if (url.pathname.startsWith("/download/")) {
      const key = url.pathname.slice(10)
      return Response.json({
        url: s3.presign(key, { expiresIn: 300 }),
      })
    }

    return new Response("Not Found", { status: 404 })
  },
})
```

### Multipart Upload

For large files, S3File automatically uses multipart upload:

```typescript
import { s3, write } from "bun"

// Large file upload (automatically uses multipart for files > 5MB)
const largeFile = Bun.file("large-video.mp4")
await write(s3.file("videos/large-video.mp4"), largeFile)
```

### Common Patterns

```typescript
import { s3, write } from "bun"

// JSON config stored in S3
const config = await s3.file("config/app.json").json()

// Streaming download
const stream = s3.file("large-data.csv").stream()
for await (const chunk of stream) {
  // process chunk
}

// Copy between S3 keys
const source = await s3.file("original.txt").text()
await write(s3.file("backup/original.txt"), source)
```
