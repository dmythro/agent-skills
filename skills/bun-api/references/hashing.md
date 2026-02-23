# Hashing Reference

## Bun.hash (Non-Cryptographic)

Fast hashing for non-security purposes: hash tables, checksums, deduplication, content fingerprinting.

```typescript
// Default: wyhash (fastest general-purpose hash)
Bun.hash(input: string | Uint8Array | ArrayBuffer): number
Bun.hash(input, seed: number): number
```

### Named Algorithms

```typescript
Bun.hash.wyhash(input: string | Uint8Array, seed?: number): number
Bun.hash.crc32(input: string | Uint8Array): number
Bun.hash.adler32(input: string | Uint8Array): number
Bun.hash.cityHash32(input: string | Uint8Array, seed?: number): number
Bun.hash.cityHash64(input: string | Uint8Array, seed?: number): bigint
Bun.hash.murmur32v3(input: string | Uint8Array, seed?: number): number
Bun.hash.murmur64v2(input: string | Uint8Array, seed?: number): bigint
```

### Algorithm Selection Guide

| Algorithm | Output | Speed | Use Case |
|-----------|--------|-------|----------|
| `wyhash` (default) | 64-bit number | Fastest | General hash tables, dedup |
| `crc32` | 32-bit number | Fast | Checksums, compatibility |
| `adler32` | 32-bit number | Fast | Checksums (used in zlib) |
| `cityHash32` | 32-bit number | Fast | Hash tables |
| `cityHash64` | 64-bit bigint | Fast | Hash tables, fingerprinting |
| `murmur32v3` | 32-bit number | Fast | Distributed systems |
| `murmur64v2` | 64-bit bigint | Fast | Distributed systems |

### Examples

```typescript
// Content deduplication
const hash = Bun.hash(await Bun.file('data.bin').arrayBuffer())

// Consistent hashing with seed
const shard = Bun.hash(key, 42) % numShards

// CRC32 checksum
const checksum = Bun.hash.crc32(content)
```

## Bun.CryptoHasher

Streaming cryptographic hash computation.

```typescript
const hasher = new Bun.CryptoHasher(algorithm: string)
```

### Algorithms

`'sha1'`, `'sha224'`, `'sha256'`, `'sha384'`, `'sha512'`, `'sha512-256'`, `'md4'`, `'md5'`, `'blake2b256'`, `'blake2b512'`, `'ripemd160'`

### Interface

```typescript
interface CryptoHasher {
  update(data: string | Uint8Array | ArrayBuffer): CryptoHasher  // Chainable
  digest(): Uint8Array
  digest(encoding: 'hex' | 'base64'): string
  copy(): CryptoHasher          // Clone current state

  readonly algorithm: string
  readonly byteLength: number   // Hash output size in bytes
}
```

### One-Shot Hashing

```typescript
// SHA-256 hex digest
new Bun.CryptoHasher('sha256').update('data').digest('hex')

// MD5 (for compatibility, not security)
new Bun.CryptoHasher('md5').update(content).digest('hex')

// Base64 output
new Bun.CryptoHasher('sha256').update('data').digest('base64')
```

### Streaming (Large Files)

```typescript
const hasher = new Bun.CryptoHasher('sha256')
const stream = Bun.file('large-file.bin').stream()

for await (const chunk of stream) {
  hasher.update(chunk)
}

const hash = hasher.digest('hex')
```

### File Checksum Helper

```typescript
async function sha256(path: string): Promise<string> {
  const hasher = new Bun.CryptoHasher('sha256')
  for await (const chunk of Bun.file(path).stream()) {
    hasher.update(chunk)
  }
  return hasher.digest('hex')
}
```

### Forking State

```typescript
const base = new Bun.CryptoHasher('sha256')
base.update('shared prefix')

const fork1 = base.copy()
fork1.update(' branch A')
const hash1 = fork1.digest('hex')

const fork2 = base.copy()
fork2.update(' branch B')
const hash2 = fork2.digest('hex')
```

## Bun.password

Password hashing with bcrypt or argon2. Replaces `bcrypt` and `argon2` npm packages.

### Async (Recommended)

```typescript
Bun.password.hash(password: string, options?: PasswordHashOptions): Promise<string>
Bun.password.verify(password: string, hash: string): Promise<boolean>
```

### Sync

```typescript
Bun.password.hashSync(password: string, options?: PasswordHashOptions): string
Bun.password.verifySync(password: string, hash: string): boolean
```

### Bcrypt (Default)

```typescript
const hash = await Bun.password.hash('password')
// Or explicitly:
const hash = await Bun.password.hash('password', {
  algorithm: 'bcrypt',
  cost: 10,              // Rounds: 4-31 (default: 10)
})

const valid = await Bun.password.verify('password', hash)  // true/false
```

### Argon2id (Recommended for New Projects)

```typescript
const hash = await Bun.password.hash('password', {
  algorithm: 'argon2id',
  memoryCost: 65536,     // KiB (default: 65536 = 64 MB)
  timeCost: 2,           // Iterations (default: 2)
})
```

### Argon2 Variants

| Algorithm | Use Case |
|-----------|----------|
| `argon2id` | Recommended default — hybrid (resists side-channel + GPU attacks) |
| `argon2i` | Side-channel resistant (use when timing attacks are the primary concern) |
| `argon2d` | GPU-resistant (faster, but vulnerable to side-channel attacks) |

### Verify Auto-Detects Algorithm

`Bun.password.verify()` automatically detects the algorithm from the hash string — you don't need to specify it:

```typescript
const bcryptHash = await Bun.password.hash('pw')
const argonHash = await Bun.password.hash('pw', { algorithm: 'argon2id' })

await Bun.password.verify('pw', bcryptHash)  // true (detects bcrypt)
await Bun.password.verify('pw', argonHash)   // true (detects argon2)
```
