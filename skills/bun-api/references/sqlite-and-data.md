# SQLite and Data Reference

## bun:sqlite

Built-in SQLite3 with zero dependencies. Faster than `better-sqlite3`.

```typescript
import { Database, Statement, constants } from 'bun:sqlite'
```

### Database Constructor

```typescript
new Database(filename: string, options?: DatabaseOptions)
```

Options:
```typescript
interface DatabaseOptions {
  readonly?: boolean        // Open read-only
  create?: boolean          // Create if not exists (default: true)
  readwrite?: boolean       // Open for reading and writing (default: true)
  strict?: boolean          // Enable strict mode for queries
}
```

### Opening Databases

```typescript
const db = new Database('mydb.sqlite')           // File-based
const db = new Database(':memory:')               // In-memory
const db = new Database('')                       // Temporary (deleted on close)
const db = new Database('mydb.sqlite', { readonly: true })

// From buffer (e.g., downloaded database)
const db = Database.open(buffer)                  // From ArrayBuffer/Uint8Array
```

### Executing SQL

```typescript
// Execute raw SQL (no return value, can be multiple statements)
db.exec('CREATE TABLE t (id INTEGER PRIMARY KEY, name TEXT)')
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    created_at TEXT DEFAULT (datetime('now'))
  );
  CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
`)

// Run with parameters
db.run('INSERT INTO users (name, email) VALUES (?, ?)', ['Alice', 'alice@example.com'])
```

### Prepared Statements

```typescript
const stmt = db.prepare(sql: string): Statement
```

#### Statement Methods

```typescript
interface Statement {
  // Execute and return results
  get(...params: any[]): any | null       // First row or null
  all(...params: any[]): any[]            // All rows
  run(...params: any[]): void             // Execute, no return
  values(...params: any[]): any[][]       // Rows as arrays (no column names)

  // Column info
  readonly columnNames: string[]
  readonly paramsCount: number

  // Lifecycle
  finalize(): void                        // Free resources (optional, GC handles it)
  toString(): string                      // SQL string
}
```

#### Parameter Binding

```typescript
// Positional (?)
const stmt = db.prepare('SELECT * FROM users WHERE id = ? AND name = ?')
stmt.get(1, 'Alice')
stmt.all(1, 'Alice')

// Indexed ($1, $2)
const stmt = db.prepare('SELECT * FROM users WHERE id = $1 AND name = $2')
stmt.get(1, 'Alice')

// Named ($name, :name, @name)
const stmt = db.prepare('SELECT * FROM users WHERE name = $name AND email = $email')
stmt.get({ $name: 'Alice', $email: 'alice@example.com' })

// Array parameter
stmt.get([1, 'Alice'])
```

### Transactions

```typescript
const insertUser = db.prepare('INSERT INTO users (name, email) VALUES ($name, $email)')

const insertMany = db.transaction((users: { name: string; email: string }[]) => {
  for (const user of users) {
    insertUser.run({ $name: user.name, $email: user.email })
  }
  return users.length
})

// Execute transaction
const count = insertMany([
  { name: 'Bob', email: 'bob@example.com' },
  { name: 'Carol', email: 'carol@example.com' },
])

// Transaction is automatically rolled back on error
// Nested transactions use savepoints

// Deferred/Immediate/Exclusive
insertMany.deferred(users)     // BEGIN DEFERRED
insertMany.immediate(users)    // BEGIN IMMEDIATE
insertMany.exclusive(users)    // BEGIN EXCLUSIVE
```

### Pragmas and Configuration

```typescript
// WAL mode (recommended for concurrent access)
db.exec('PRAGMA journal_mode = WAL')

// Performance tuning
db.exec('PRAGMA synchronous = NORMAL')
db.exec('PRAGMA cache_size = -64000')   // 64MB cache
db.exec('PRAGMA temp_store = MEMORY')

// Foreign keys
db.exec('PRAGMA foreign_keys = ON')

// Query pragmas
const mode = db.prepare('PRAGMA journal_mode').get()
```

### Database Methods

```typescript
interface Database {
  exec(sql: string): void
  prepare(sql: string): Statement
  run(sql: string, params?: any[]): void
  query(sql: string): Statement          // Alias for prepare
  transaction(fn: Function): Function
  close(): void
  serialize(): Uint8Array                // Serialize database to buffer

  readonly filename: string
  readonly inTransaction: boolean
}
```

### Column Type Mapping

| SQLite Type | JavaScript Type |
|---|---|
| `INTEGER` | `number` (or `bigint` if > 2^53) |
| `REAL` | `number` |
| `TEXT` | `string` |
| `BLOB` | `Uint8Array` |
| `NULL` | `null` |

## Bun.hash (Non-Cryptographic)

Fast hashing for non-security purposes (hash tables, checksums, dedup).

```typescript
// Default: wyhash (fastest)
Bun.hash(input: string | Uint8Array | ArrayBuffer): number
Bun.hash(input, seed: number): number

// Named algorithms
Bun.hash.wyhash(input: string | Uint8Array, seed?: number): number
Bun.hash.crc32(input: string | Uint8Array): number
Bun.hash.adler32(input: string | Uint8Array): number
Bun.hash.cityHash32(input: string | Uint8Array, seed?: number): number
Bun.hash.cityHash64(input: string | Uint8Array, seed?: number): bigint
Bun.hash.murmur32v3(input: string | Uint8Array, seed?: number): number
Bun.hash.murmur64v2(input: string | Uint8Array, seed?: number): bigint
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
  readonly byteLength: number   // Hash output size
}
```

### Examples

```typescript
// One-shot
const hash = new Bun.CryptoHasher('sha256')
  .update('hello')
  .digest('hex')

// Streaming (process large file)
const hasher = new Bun.CryptoHasher('sha256')
const stream = Bun.file('large-file.bin').stream()
for await (const chunk of stream) {
  hasher.update(chunk)
}
const hash = hasher.digest('hex')

// File checksum helper
async function sha256(path: string): Promise<string> {
  const hasher = new Bun.CryptoHasher('sha256')
  for await (const chunk of Bun.file(path).stream()) {
    hasher.update(chunk)
  }
  return hasher.digest('hex')
}
```

## Bun.password

Password hashing with bcrypt or argon2.

```typescript
// Async (recommended)
Bun.password.hash(password: string, options?: PasswordHashOptions): Promise<string>
Bun.password.verify(password: string, hash: string): Promise<boolean>

// Sync
Bun.password.hashSync(password: string, options?: PasswordHashOptions): string
Bun.password.verifySync(password: string, hash: string): boolean
```

### Options

```typescript
// Bcrypt (default)
{
  algorithm: 'bcrypt',
  cost: 10,              // Rounds (4-31, default: 10)
}

// Argon2id
{
  algorithm: 'argon2id',
  memoryCost: 65536,     // KiB (default: 65536 = 64MB)
  timeCost: 2,           // Iterations (default: 2)
}

// Argon2i
{
  algorithm: 'argon2i',
  memoryCost: 65536,
  timeCost: 2,
}

// Argon2d
{
  algorithm: 'argon2d',
  memoryCost: 65536,
  timeCost: 2,
}
```

## bun:jsc (Serialization)

Binary structured clone for efficient serialization of complex objects.

```typescript
import { serialize, deserialize } from 'bun:jsc'

// Serialize to binary
const bytes: Uint8Array = serialize(value: any)

// Deserialize from binary
const value: any = deserialize(bytes: Uint8Array)
```

### Supported Types (beyond JSON)

- `Date`, `RegExp`, `Map`, `Set`
- `ArrayBuffer`, `SharedArrayBuffer`, typed arrays
- `Error` objects
- `Blob`, `File`
- Circular references
- `undefined` (preserved, not converted to null)

### Use Cases

- IPC message passing (faster than JSON for complex structures)
- Caching computed objects to disk
- Transferring data between workers
