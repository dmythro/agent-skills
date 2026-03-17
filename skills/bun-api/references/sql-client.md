# SQL Client Reference (Bun.sql)

Built-in SQL client for PostgreSQL, MySQL, and SQLite. Zero dependencies, tagged template literals, automatic prepared statements, connection pooling.

## Prerequisites

A project can use `Bun.sql()` when it has a database connection URL:
- `DATABASE_URL` environment variable (in `.env`, `.env.local`, or shell)
- Direct URL string passed to `new SQL()`
- Explicit connection options object

## Importing

```typescript
import { sql, SQL } from "bun"
```

- `sql` -- default instance, auto-connects using `DATABASE_URL` from environment
- `SQL` -- constructor for explicit connection configuration

## Auto-Detection

The adapter is detected from the connection URL:

| URL pattern | Adapter |
|---|---|
| `postgres://`, `postgresql://` | PostgreSQL |
| `mysql://`, `mysql2://` | MySQL |
| `sqlite://`, `file://`, `:memory:` | SQLite |
| Anything else | PostgreSQL (default) |

## Basic Queries

```typescript
// Uses DATABASE_URL from environment
const users = await sql`SELECT * FROM users WHERE active = ${true} LIMIT ${10}`

// Result is an array of objects
for (const user of users) {
  console.log(user.name, user.email)
}
```

Parameters are automatically escaped -- **never use string interpolation** for values.

## Explicit Connection

```typescript
// PostgreSQL
const pg = new SQL("postgres://user:pass@localhost:5432/mydb")

// MySQL
const mysql = new SQL("mysql://user:pass@localhost:3306/mydb")

// SQLite
const sqlite = new SQL("sqlite://myapp.db")

// With options
const db = new SQL({
  url: "postgres://user:pass@localhost:5432/mydb",
  max: 20,
  idleTimeout: 30,
  maxLifetime: 0,
  connectionTimeout: 30,
  tls: true,
})
```

## Connection Options

```typescript
interface SQLOptions {
  url?: string              // Connection URL
  hostname?: string         // Database host
  port?: number             // Database port
  database?: string         // Database name
  username?: string         // Username
  password?: string | (() => string)  // Password or resolver function
  max?: number              // Max connections in pool (default: varies)
  idleTimeout?: number      // Close idle connections after N seconds
  maxLifetime?: number      // Connection lifetime in seconds (0 = forever)
  connectionTimeout?: number // Timeout for new connections (seconds)
  tls?: boolean | TLSOptions // Enable TLS or detailed TLS config
  prepare?: boolean         // Named prepared statements (default: true)
  onconnect?: (client: any) => void
  onclose?: (client: any) => void
}
```

## Insert, Update, Delete

### Insert with Object Helper

```typescript
const user = { name: "Alice", email: "alice@example.com" }

// Expands to: INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')
const [newUser] = await sql`INSERT INTO users ${sql(user)} RETURNING *`

// Select specific columns from object
await sql`INSERT INTO users ${sql(user, "name", "email")}`
```

### Bulk Insert

```typescript
const users = [
  { name: "Alice", email: "alice@example.com" },
  { name: "Bob", email: "bob@example.com" },
]
await sql`INSERT INTO users ${sql(users)}`
```

### Update with Object Helper

```typescript
const updates = { name: "Alice Smith", email: "alice.smith@example.com" }

// Update all fields from object
await sql`UPDATE users SET ${sql(updates)} WHERE id = ${userId}`

// Update specific columns only
await sql`UPDATE users SET ${sql(updates, "name", "email")} WHERE id = ${userId}`
```

### Delete

```typescript
await sql`DELETE FROM users WHERE id = ${userId}`
```

## Transactions

```typescript
await sql.begin(async (tx) => {
  const [user] = await tx`
    INSERT INTO users (name, email) VALUES (${"Alice"}, ${"alice@example.com"})
    RETURNING *
  `
  await tx`
    INSERT INTO audit_log (action, user_id) VALUES ('created', ${user.id})
  `
})
// Automatically committed on success, rolled back on error
```

### Savepoints

```typescript
await sql.begin(async (tx) => {
  await tx`INSERT INTO users (name) VALUES (${"Alice"})`

  await tx.savepoint(async (sp) => {
    await sp`UPDATE users SET status = 'active'`
    if (someCondition) {
      throw new Error("Rollback to savepoint")
    }
  })

  // Continues even if savepoint rolled back
  await tx`INSERT INTO audit_log (action) VALUES ('user_created')`
})
```

## Connection Pool

Connections are created lazily and reused automatically.

```typescript
const db = new SQL("postgres://...")  // No connections created yet

await db`SELECT 1`  // First connection created
await db`SELECT 2`  // Connection reused

// Concurrent queries use multiple connections
await Promise.all([
  db`INSERT INTO users ${sql({ name: "Alice" })}`,
  db`UPDATE users SET name = ${"Bob"} WHERE id = ${1}`,
])
```

### Reserve Exclusive Connection

```typescript
const reserved = await sql.reserve()
try {
  await reserved`INSERT INTO users (name) VALUES (${"Alice"})`
  await reserved`SELECT * FROM users`
} finally {
  reserved.release()
}

// Or with Symbol.dispose
{
  using reserved = await sql.reserve()
  await reserved`SELECT 1`
} // Automatically released
```

### Close Pool

```typescript
await sql.close()                   // Wait for queries to finish, then close
await sql.close({ timeout: 5 })    // Wait 5 seconds max
await sql.close({ timeout: 0 })    // Close immediately
```

## MySQL-Specific

```typescript
const mysql = new SQL("mysql://user:pass@localhost:3306/mydb")

// Same tagged template interface
const users = await mysql`SELECT * FROM users WHERE active = ${true}`

// MySQL connection formats
new SQL("mysql://user:pass@localhost:3306/database")
new SQL("mysql2://user:pass@localhost:3306/database")  // mysql2 compat
new SQL("mysql://user:pass@localhost/db?ssl=true")
new SQL("mysql://user:pass@/db?socket=/var/run/mysqld/mysqld.sock")
```

MySQL support includes prepared statements, binary protocol, multiple result sets, caching_sha2_password auth, SSL/TLS, and query pipelining.

## Prepared Statements

By default, queries are automatically prepared and cached for performance. Disable with `prepare: false` if needed (PGBouncer in transaction mode, dynamic SQL).

```typescript
const db = new SQL({
  url: "postgres://...",
  prepare: false,  // Use unnamed prepared statements
})
```
