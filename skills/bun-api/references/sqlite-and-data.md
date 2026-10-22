# SQLite and Data Reference

## bun:sqlite

Built-in SQLite3 with zero dependencies. Faster than `better-sqlite3`. Bundles SQLite 3.53.4 on
Linux and Windows (as of Bun v1.4.3); on macOS it loads the system SQLite (3.54.0 on macOS 27),
which `Database.setCustomSQLite(path)` replaces before the first `new Database()`.

**Changed in 1.4.3:** statements with more than 65535 parameters bind correctly -- a 70,000-value
`IN (...)` threw `expected 4464 values, received 70000`, and named parameters past the limit were
silently bound as `NULL`. `Database.deserialize()` accepts an `ArrayBuffer`, a detached
TypedArray binds an empty BLOB (was `NULL`), and a 2 GiB `Uint8Array` throws `SQLITE_TOOBIG`.
A Worker may repeat `setCustomSQLite()` with the path the process already loaded.

> Full API: `node_modules/bun-types/docs/runtime/sqlite.mdx`.

**Changed in 1.4.4** (verified against 1.4.3 unless marked):

- `db.function()`, `db.aggregate()`, and `db.onInsert` / `onUpdate` / `onDelete` are new (below),
  as are `stmt.reader`, `stmt.readonly`, and `db[Symbol.for("sqlite-type")] === "bun:sqlite"`.
- A runtime error in any statement of a `db.run("a; b; c")` / `db.exec()` string throws there and
  skips the rest. 1.4.3 threw only for the last statement: a `UNIQUE` violation in the middle was
  swallowed and the remaining statements ran.
- Named parameters bind with or without the `$` / `:` / `@` prefix in both modes. Without
  `strict: true`, 1.4.3 bound `NULL` for `{ name: 1 }`; with it, 1.4.3 threw
  `Missing parameter "name"` for `{ $name: 1 }`. When an object has both keys, the prefixed one
  wins -- the bare one with `strict: true`.
- Binding a `Date`, `Map`, or `Promise` throws `TypeError` (1.4.3 bound `NULL`), and
  `{ readonly: true, create: true }` throws `TypeError` (1.4.3 opened read-write).
- `db.transaction(fn)` rolls back when `fn` returns an already-rejected promise, as an `async`
  function throwing before its first `await` does; 1.4.3 committed. `changes` counts only the
  statement's own rows, not trigger, cascade, or FTS5 shadow-table writes (1.4.3 counted them),
  and `row["1"]` works for `SELECT 1`.
- Release notes: an existing `Statement` sees new column names after `ALTER TABLE`; `break` out of
  `stmt.iterate()` closes the cursor, so a later `DROP TABLE` no longer fails with
  `database table is locked`; `stmt.get()` stops after the first row; unread `iterate()` over a
  writing statement commits at exit; `db.query()` stops caching SQL text over 64 KB; and
  `PRAGMA hard_heap_limit` / `soft_heap_limit` are accepted but no longer enforced.

**Changed in 1.4 -- `db.close()` finalizes statements.** `close()` now finalizes every
`db.query()` statement, not only cached ones; it previously threw `database is locked`.
A statement finalized by `close()` throws when used afterwards. `db.prepare()` statements keep
working until finalized -- `db.close(true)` finalizes those too, releases the connection
immediately, and throws if SQLite fails to close it. `using db = new Database(...)` calls
`close(true)`. A column aliased `AS ""` is
now kept in row objects and `stmt.columnNames` (it was dropped, and a trailing one made
`.all()` return a number); `columnNames` throws after `finalize()`.

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
  create?: boolean          // Create if not exists (default: true); false throws for a missing file
  readwrite?: boolean       // Open for reading and writing (default: true)
  strict?: boolean          // Throw on a missing parameter instead of binding NULL
  safeIntegers?: boolean    // INTEGER columns as bigint
}
```

Contradictory options such as `{ readonly: true, create: true }` throw `TypeError` (v1.4.4+).

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

A `Statement` is one SQL statement: `prepare()` and `query()` compile the first statement of the
string and ignore the rest. Run a string of several statements with `db.run()` or `db.exec()`.

#### Statement Methods

```typescript
interface Statement {
  // Execute and return results
  get(...params: any[]): any | null       // First row or null; stops after the first row (v1.4.4+)
  all(...params: any[]): any[]            // All rows
  run(...params: any[]): { lastInsertRowid: number | bigint; changes: number }
  values(...params: any[]): any[][]       // Rows as arrays (no column names)
  iterate(...params: any[]): IterableIterator<any>   // One row at a time
  safeIntegers(enabled?: boolean): this | boolean     // Per-statement bigint switch

  // Column info
  readonly columnNames: string[]
  readonly paramsCount: number
  readonly reader: boolean                // Returns rows (v1.4.4+)
  readonly readonly: boolean              // Makes no direct change to the database (v1.4.4+)

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

// Named ($name, :name, @name) -- keys with or without the prefix (v1.4.4+)
const stmt = db.prepare('SELECT * FROM users WHERE name = $name AND email = $email')
stmt.get({ $name: 'Alice', $email: 'alice@example.com' })
stmt.get({ name: 'Alice', email: 'alice@example.com' })   // 1.4.3: bare keys only with strict: true

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

**Keep the transaction function synchronous.** The driver commits when the function returns, and
an `async` function returns at its first `await`: everything after it runs outside the
transaction, and an error thrown there rolls nothing back. Do the awaited work first and pass the
results in. A function that returns an already-rejected promise (an `async` function throwing
before its first `await`) rolls back as of v1.4.4; 1.4.3 committed it.

### User-Defined Functions (v1.4.4+)

```typescript
db.function('add2', (a, b) => a + b)                    // arity = fn.length; returns the Database
db.query('SELECT add2(12, 4) AS sum').get()              // { sum: 16 }

db.function('count_args', { varargs: true }, (...args) => args.length)

db.aggregate('add_all', {
  start: 0,                                              // or () => [] for a fresh object per group
  step: (total, value) => total + value,                 // returning undefined keeps the accumulator
  // result: acc => ...,                                 // final value; default is the accumulator
  // inverse: (acc, value) => ...,                       // makes it usable as a window function
})
db.query('SELECT category, add_all(dollars) AS total FROM expenses GROUP BY category').all()
```

| Option | Effect |
|---|---|
| `varargs` | Any number of arguments |
| `deterministic` | Same arguments, same result -- allows the function in indexes and generated columns |
| `directOnly` | Only SQL you pass may call it, not views or triggers -- use for side effects on untrusted database files |
| `safeIntegers` | `INTEGER` arguments arrive as `bigint` |

- A function with the same name and arity replaces the earlier one, SQLite built-ins included.
- What the function throws ends the query and is rethrown by the `get()` / `all()` / `run()` that
  ran it. Returning an object or a promise throws `TypeError`; `undefined` is `NULL`.
- **No queries inside a function**: any use of the same `Database` throws
  `TypeError: This database connection is busy executing a query`. A second `Database` works.
- The signatures match better-sqlite3.

### Update Hooks (v1.4.4+)

```typescript
db.onInsert = (db, tableName, rowid, schemaName) => {
  queueMicrotask(() => {
    // the statement has ended: reading the new row is allowed here
  })
}
db.onUpdate = (db, tableName, rowid) => {}
db.onDelete = (db, tableName, rowid) => {}
db.onInsert = null                                         // remove
```

One `sqlite3_update_hook` underneath, so its rules apply: the hook runs during the statement,
before commit (a rolled-back change was still reported); it cannot use its `Database` (defer with
`queueMicrotask()`); a throw is reported as an uncaught exception and the statement goes on; and
`WITHOUT ROWID` tables, rows that `INSERT OR REPLACE` removes, other connections' writes, and an
unconditional `DELETE` on a trigger-less table are not reported. `rowid` is a `bigint` with
`safeIntegers: true`.

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
  run(sql: string, params?: any[]): { lastInsertRowid: number | bigint; changes: number }
  query(sql: string): Statement          // Cached prepare (the 20 most recent SQL strings)
  transaction(fn: Function): Function
  close(throwOnError?: boolean): void    // true: finalize every outstanding statement now
  serialize(): Uint8Array                // Serialize database to buffer
  function(name: string, fn: Function): this                    // v1.4.4+
  function(name: string, options: FunctionOptions, fn: Function): this
  aggregate(name: string, options: AggregateOptions): this      // v1.4.4+
  onInsert: UpdateHook | null                                   // v1.4.4+, also onUpdate / onDelete

  readonly filename: string
  readonly inTransaction: boolean
}
```

### Column Type Mapping

| SQLite Type | JavaScript Type |
|---|---|
| `INTEGER` | `number` (`bigint` with `safeIntegers`). Binding: a safe-integer `number` goes in as `INTEGER`, any other as `REAL` |
| `REAL` | `number` |
| `TEXT` | `string` |
| `BLOB` | `Uint8Array` |
| `NULL` | `null` |

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
