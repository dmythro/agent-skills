# SQL Client Reference (Bun.sql)

Built-in SQL client for PostgreSQL, MySQL, and SQLite. Zero dependencies, tagged template literals, automatic prepared statements, connection pooling.

> Full API: `node_modules/bun-types/docs/runtime/sql.mdx`.

**Changed in 1.4 -- date and JSON decoding.** Two changes silently alter values read from
MySQL/MariaDB, so audit any offset-correction or `JSON.parse()` you added as a workaround:

- MySQL `DATETIME` and `TIMESTAMP` decode as **UTC**, matching how `Bun.sql` encodes them, so
  a `Date` round-trips unchanged. Before 1.4 they came back shifted by the host's UTC offset
  on any machine not running in UTC. Postgres `timestamp` read through `.simple()` is UTC too;
  `timestamptz` is unaffected.
- MariaDB 10.5+ `JSON` columns and JSON function results (`JSON_OBJECT()`, `JSON_EXTRACT()`)
  are **parsed into objects**. They returned JSON text before -- remove the `JSON.parse()`.

Also new or changed in 1.4: Postgres `infinity`/`-infinity` **dates/timestamps** decode to
the numbers `Infinity` and `-Infinity` instead of an invalid `Date`; `PGSSLMODE` is honored from the environment
(`PGSSLMODE=require` against a non-TLS server now fails rather than connecting in plaintext);
`connectionTimeout` bounds the entire handshake rather than restarting per packet; and
`sql.unsafe()` / `sql.file()` accept an object of named parameters for `:name`, `$name`, and
`@name` placeholders on SQLite (an object previously bound nothing and returned no rows).
Float8/float4 read as **text** (`.simple()`, `sql.unsafe()`) decoded `'Infinity'`/`'-Infinity'`
to `NaN` on 1.4.0; fixed in 1.4.1 (oven-sh/bun#39777). Also fixed in 1.4.1: `timestamptz` with a
seconds-level offset returned `Invalid Date` in text queries and arrays, a `Date` written to a
MySQL `TIMESTAMP` column landed at the wrong instant under a non-UTC session time zone,
`close()`/`ref()`/`unref()` crashed after the server dropped the connection, and `sql.close()`
fired `onclose` for pool connections that never connected.

**Changed in 1.4.3** (from the release notes):

- **`sql.begin()` rejects when Postgres rolled the transaction back.** A callback that caught a
  failed query's error used to resolve while `COMMIT` was answered with `ROLLBACK`; it now
  rejects with `ERR_POSTGRES_COMMIT_ROLLED_BACK`, and nothing was committed. Let errors
  propagate, or use `tx.savepoint()` for the step that may fail.
- **One bad value fails one query.** A parameter that throws while encoding, or a result value
  the client cannot decode (a multidimensional array, MariaDB `JSON` that `JSON.parse` refuses),
  rejects only that query; it closed the connection and failed every pipelined query and the
  open transaction. A write with `RETURNING` may already be stored when its decode fails -- do
  not retry it blindly; cast the column to `text` or use `.raw()` to read it.
- **Postgres `bytea` rejects non-binary values.** Binding a `number[]`, a plain object, or a
  `Date` throws `TypeError`; it silently stored an empty `bytea`.
- **`tls: Bun.file("ca.pem")`** (or `ssl:`) is the CA to verify against with `verify-full`, the
  same as `tls: { ca: Bun.file("ca.pem") }` -- the shorthand for an RDS or Cloud SQL bundle.
- A `numeric(p, s)` zero reads as `"0.0000"`, not `"0"`, through prepared statements.
- Correctness fixes worth an upgrade: after a failed pipelined Postgres query, a statement new to
  that connection could resolve with another query's rows (every time behind pgdog), and a MySQL
  query that failed client-side in a connection's queue never settled while the next one got its
  rows.

**Changed in 1.4.4.** Verified on the SQLite adapter; PostgreSQL and MySQL items come from the
release notes and the shipped `sql.mdx`:

- **`PASSWORD` is no longer read** from the environment. Use `PGPASSWORD`, `MYSQL_PASSWORD`,
  `MARIADB_PASSWORD`, the URL, or the `password` option.
- **`new SQL({ filename })` selects SQLite** without `adapter`; 1.4.3 connected to PostgreSQL
  (`ERR_POSTGRES_CONNECTION_REFUSED`). A bare connection string such as `"app.db"` still needs
  `{ adapter: "sqlite" }`, since Bun reads it as a PostgreSQL host.
- **A SQLite string of several statements runs all of them** (`sql.unsafe()`, `sql.file()`), and
  resolves to the first statement's rows. 1.4.3 ran only the first statement when it returned rows,
  so a migration starting with a `PRAGMA` silently did nothing else. Only PostgreSQL wraps such a
  string in an implicit transaction; on MySQL and SQLite the statements before a failing one stay
  applied.
- **Result metadata:** `affectedRows` is the number of rows written (`0` for a `SELECT`) on SQLite
  and PostgreSQL, where it was always `null`; `count` is right for `INSERT ... SELECT`; a
  `sql.begin()` / `tx.savepoint()` callback's result keeps `count` and `command`.
- **Transactions on SQLite queue:** a second `sql.begin()` waits instead of failing with
  `cannot start a transaction within a transaction`, and a failed `COMMIT` no longer leaves the
  transaction open (release notes).
- **Cancelling:** `query.cancel()` before the query is sent rejects it with
  `ERR_SQLITE_QUERY_CANCELLED` / `ERR_POSTGRES_QUERY_CANCELLED` / `ERR_MYSQL_QUERY_CANCELLED`; on
  1.4.3 such a query never settled.
- **`sql instanceof Bun.SQL`** is `true`; it threw `TypeError`.
- **PostgreSQL (release notes):** JavaScript arrays bind to array parameters; 35 more array types
  (`uuid[]`, `int4range[]`, `tsvector[]`) decode as arrays instead of `"{...}"`; `sql.array(["a"])`
  no longer casts to `JSON[]`, and its `null` elements are SQL `NULL`; a `Date` bound to `text`,
  `date`, or `time` is ISO 8601; integers from 2^51 to 2^53 - 1 go out as `int8` and come back as
  strings; an Invalid Date rejects; plain `COPY` without a stream rejects with
  `ERR_POSTGRES_COPY_NOT_SUPPORTED`; cached statements are re-prepared after `ALTER TABLE` /
  `DISCARD ALL`, and each connection keeps its 1,024 most recent; `sslmode=prefer` falls back to
  plaintext against a server without TLS; `PGUSERNAME`, `PGAPPNAME`, `PGOPTIONS`, and
  `PGCONNECT_TIMEOUT` are read.
- **MySQL and MariaDB (release notes):** queries on one connection are pipelined again (broken
  since 1.2.22), `caching_sha2_password` without TLS works for passwords of 20 bytes or more and no
  longer needs `allowPublicKeyRetrieval: true` on every login, `lastInsertRowid` above 2^53 - 1 is a
  string (a `BigInt` with `bigint: true`), MariaDB accepts object and array parameters, a refused
  connection throws the server's error (`errno` 1040), and `sql("db.table")` quotes each part.
- **Pool (release notes):** `idleTimeout` and `maxLifetime` let a busy, reserved, or in-transaction
  connection finish first; a pool that lost connections grows back to `max`; `sql.begin()` and
  `sql.reserve()` wait their turn (FIFO); sockets use 60 s TCP keepalive; and
  `sql.close({ timeout: 0 })` closes at once.
- **`bun --hot` reuses connections:** a `new SQL()` with the same settings picks up the previous
  module's connections after a reload, keeping session state (`SET`, temp tables) and running
  `onconnect` again; 1.4.3 opened up to `max` new ones per save and left the old ones open
  (release notes).

For PgBouncer transaction pooling, construct with `new Bun.SQL({ prepare: false })` -- since
v1.3.11 Bun sends each such query's Parse+Bind+Execute as one atomic batch, so PgBouncer can
no longer split it across connections and return the wrong result.

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
| `mariadb://` | MariaDB (the MySQL driver; reads `MARIADB_*` variables where MySQL reads `MYSQL_*`) |
| `sqlite://`, `sqlite:`, `file://`, `file:`, `:memory:` | SQLite |
| Anything else, `localhost:5432/mydb` included | PostgreSQL (default) |

The `filename` option selects SQLite on its own as of v1.4.4. For the other settings, an option
wins over the URL you pass, which wins over a URL from the environment (`DATABASE_URL`), which wins
over per-field variables (`PGHOST`, `MYSQL_USER`); `PGDATABASE` / `MYSQL_DATABASE` /
`MARIADB_DATABASE` are the exception and beat the URL's database. An options object without a
`url` changes single fields of the environment URL -- `new SQL({ hostname: "replica.internal" })`
keeps `DATABASE_URL`'s user, password, and `sslmode`.

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
  adapter?: "postgres" | "mysql" | "mariadb" | "sqlite"
  filename?: string         // SQLite file; selects SQLite without `adapter` (v1.4.4+)
  transform?: Transform | NameTransform  // SQL.camel / SQL.pascal / SQL.kebab, or { column, undefined } (v1.4.4+)
  channelBinding?: "disable" | "prefer" | "require"   // PostgreSQL SCRAM-SHA-256-PLUS (v1.4.4+)
  foundRows?: boolean       // MySQL: affectedRows counts matched rows, like mysql2 (v1.4.4+)
}
```

PostgreSQL and MySQL URLs also take `?connect_timeout=`, `?idle_timeout=`, `?max_lifetime=`, and
`?max=` (v1.4.4+), and PostgreSQL URLs libpq's `host`, `port`, `user`, `password`, `dbname`,
`sslrootcert`, `sslcert`, `sslkey`, and `sslpassword` parameters, which 1.4.3 forwarded to the
server and got refused. `tls.checkServerIdentity(hostname, cert)` replaces the hostname check on
PostgreSQL, MySQL, and MariaDB and upgrades a lower SSL mode to `verify-full` (release notes):

```typescript
const PINNED = "AB:CD:..."   // SHA-256 fingerprint of the expected server certificate
const db = new SQL("postgres://app@db.example.com/app?connect_timeout=5", {
  channelBinding: "require",
  tls: {
    ca: Bun.file("./ca.pem"),
    checkServerIdentity: (hostname, cert) =>
      cert.fingerprint256 === PINNED ? undefined : new Error("pin mismatch"),
  },
})
```

## Column Name Transforms (v1.4.4+)

```typescript
const sql = new SQL({ transform: SQL.camel })

await sql`INSERT INTO users ${sql({ firstName: "Alice", createdAt: new Date() })}`
// INSERT INTO users ("first_name", "created_at") VALUES ($1, $2)
const [user] = await sql`SELECT first_name, created_at FROM users`
user.firstName
```

| Preset | `user_id` in code | Results only | Helpers only |
|---|---|---|---|
| `SQL.camel` | `userId` | `SQL.toCamel` | `SQL.fromCamel` |
| `SQL.pascal` | `UserId` | `SQL.toPascal` | `SQL.fromPascal` |
| `SQL.kebab` | `user-id` | `SQL.toKebab` | `SQL.fromKebab` |

- Presets work on every adapter (verified on SQLite) and match postgres.js names, but there is no
  `transform.value` / `transform.row`: keys inside `json` / `jsonb` values are not renamed.
- Custom: `transform: { column: { from: name => ..., to: name => ... } }`. `from` renames row keys
  and `result.columns` names; `to` renames `sql(object)` keys and every `sql("name")`, table
  names included. Bun calls `from` once per statement column, so it must be deterministic.
- `transform: { undefined: null }` sends `undefined` as that value, and the insert and update
  helpers then keep every column. Without it, an `undefined` value is `NULL` -- and a column that
  is `undefined` in every row is left out, so its default applies.

## Parameters by Type

| JavaScript value | PostgreSQL | MySQL / MariaDB | SQLite |
|---|---|---|---|
| `Date` | Instant for `timestamp` / `timestamptz`; ISO 8601 for `date`, `time`, `text` | `DATETIME` in UTC | Not supported -- pass `date.toISOString()` or `date.getTime()` |
| Plain object | JSON | JSON | Not supported -- pass `sql.json(value)` |
| `Array` | Array literal for built-in array types (v1.4.4+); JSON for `json` / `jsonb` | JSON | Not supported -- pass `sql.json(value)` |
| `Buffer`, typed arrays | `bytea` | Binary | `BLOB` |
| `string` | A JSON string for `json` / `jsonb`; as is otherwise | As is | `TEXT` |
| `null`, `undefined` | `NULL` | `NULL` | `NULL` |

On PostgreSQL with prepared statements, pass the object itself to a `json` / `jsonb` column, not
`JSON.stringify(object)` -- a string is stored as a JSON string. `sql.json(value)` (v1.4.4+)
stores any value as JSON on every adapter, the way to store a number, boolean, or JSON `null`
(a bare `null` is SQL `NULL`):

```typescript
await sql`INSERT INTO settings (data) VALUES (${sql.json(42)})`      // 42
await sql`INSERT INTO settings (data) VALUES (${sql.json("dark")})`  // "dark"
await sql`INSERT INTO settings (data) VALUES (${sql.json(null)})`    // JSON null
await sql`INSERT INTO people ${sql({ moods: sql.array(["ok", "happy"], "mood") })}`  // enum[]: name the type
```

Bun reads parameters when it sends the query, not when the template is evaluated: do not mutate a
`Buffer`, `Date`, or object you passed until the query settles.

An empty list matches nothing: `WHERE id IN ${sql([])}` becomes `IN (NULL)` on PostgreSQL and
MySQL (1.4.3: a syntax error there), and `IN ()` on SQLite -- where `NOT IN ()` matches every row.

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

## Result Metadata (v1.4.4+)

The array a query resolves to carries `count`, `command`, `lastInsertRowid`, and `affectedRows`
(enumerable) plus `columns` and `statement` (not enumerable; `JSON.stringify` sees only rows).

| Property | PostgreSQL | MySQL / MariaDB | SQLite |
|---|---|---|---|
| `count` | Rows returned, or written when none are returned | same | same |
| `affectedRows` | Rows written; `0` for reads | same | same (was `null` on PostgreSQL and SQLite before 1.4.4) |
| `lastInsertRowid` | `null` -- use `RETURNING` | First `AUTO_INCREMENT` value | `rowid` of the last insert |
| `command` | Command tag (`"INSERT"`) | `null` | First keyword |
| `columns` | `{ name, type: <OID>, table, number }[]` | `{ name, type, table, length, flags }[]` | `null` |
| `statement` | `{ string, types, columns }` | `{ string, types, columns }` | `null` |

`columns[i].type` tells apart columns that decode to the same JavaScript value, such as `jsonb`
(OID 3802) and `text[]` (1009). `columns` lists duplicate names a row object cannot hold, and is
`[]` for a statement that returns no rows. Treat `statement` as read-only: runs of one prepared
statement share it.

## Cancelling Queries (v1.4.4+)

A query starts on `.then()` / `.catch()` / `.finally()` / `.execute()` -- `await` calls `.then()`
-- and one that is never awaited or executed never runs. `query.active` is `true` while a
connection runs it; `query.cancelled` is `true` after `cancel()`.

```typescript
const query = sql`SELECT pg_sleep(30)`.execute()
setTimeout(() => query.cancel(), 100)
await query   // rejects: canceling statement due to user request
```

| The query | PostgreSQL | MySQL / MariaDB | SQLite |
|---|---|---|---|
| Not sent yet | Never runs; `ERR_POSTGRES_QUERY_CANCELLED` | Never runs; `ERR_MYSQL_QUERY_CANCELLED` | Never runs; `ERR_SQLITE_QUERY_CANCELLED` |
| Running on the server | Bun opens a second connection to cancel it; rejects with the server's error (`errno` `"57014"`) | Runs to the end and settles with its result | Runs inside the call that starts it; cannot be cancelled |

An `ERR_POSTGRES_QUERY_CANCELLED` without a `hint` never reached the server and is safe to retry.
`cancel()` existed on 1.4.3 but did nothing; a query cancelled before it was sent never settled.

A query started with `.execute()` and never awaited loses some failures silently -- add a
`.catch()` to fire-and-forget queries:

```typescript
sql`UPDATE stats SET hits = hits + 1`.catch(err => console.error(err))
```

## COPY Streams (PostgreSQL, v1.4.4+)

`query.writable()` feeds `COPY ... FROM STDIN` and `query.readable()` reads `COPY ... TO STDOUT`,
both as web streams with backpressure, carrying raw bytes in the statement's format (`text`,
`csv`, or `binary`). Memory stays flat whatever the table size (release notes: 1,000 MiB at
255 MiB/s with a 59 MiB peak RSS).

```typescript
// Load a CSV file into a table
await Bun.file("users.csv").stream().pipeTo(sql`COPY users (id, name) FROM STDIN (FORMAT csv)`.writable())

// Stream a table as an HTTP response
Bun.serve({
  routes: {
    "/users.csv": () => new Response(sql`COPY users TO STDOUT (FORMAT csv, HEADER)`.readable()),
  },
})
```

`COPY` as a plain query, without a stream, rejects with `ERR_POSTGRES_COPY_NOT_SUPPORTED`.

## Several Statements in One String

`sql.unsafe()` and `sql.file()` send their text as written, with the database's own placeholders:
`$1` on PostgreSQL, `?` on MySQL, and `?`, `?1`, `$1`, `:name`, `$name`, `@name` on SQLite, where
an object of named parameters binds with or without the prefix (v1.4.4+).

| Adapter | Result | Atomic? |
|---|---|---|
| PostgreSQL | An array with one result per statement | Yes -- one implicit transaction |
| MySQL | An array with one result per statement | No -- statements before a failure stay applied |
| SQLite | One result: the first statement's rows, or the total `count` written | No -- wrap in `sql.begin()` |

On SQLite only the first statement may have parameters; a later one with a placeholder fails when
Bun reaches it. Before 1.4.4, SQLite stopped after a first statement that returned rows.
