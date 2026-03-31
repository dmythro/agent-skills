# Redis Client Reference

## Bun.redis

Built-in Redis client with zero dependencies. Works with Redis and Valkey servers.

### Connection

```typescript
import { redis, RedisClient } from "bun"

// Default client -- reads from environment variables:
// REDIS_URL, VALKEY_URL, or defaults to redis://localhost:6379
await redis.set("key", "value")
const value = await redis.get("key")

// Custom client with connection string
const clientFromUrl = new RedisClient("redis://username:password@localhost:6379")

// Custom client with options
const clientWithOptions = new RedisClient({
  url: "redis://localhost:6379",
  // Or individual fields:
  hostname: "localhost",
  port: 6379,
  username: "default",
  password: "secret",
  tls: true,
  db: 0,
})
```

### String Commands

```typescript
await redis.set("key", "value")
await redis.get("key")                        // "value" | null
await redis.del("key")                        // number of keys deleted
await redis.exists("key")                     // 0 | 1

// With expiration
await redis.set("session", "data", "EX", 3600)  // expires in 1 hour
await redis.set("temp", "data", "PX", 5000)     // expires in 5 seconds (ms)

// Set if not exists
await redis.setnx("lock", "holder")

// Increment/decrement
await redis.incr("counter")                   // increment by 1
await redis.incrby("counter", 5)              // increment by N
await redis.decr("counter")
await redis.decrby("counter", 3)

// Multiple keys
await redis.mget("key1", "key2", "key3")      // (string | null)[]
await redis.mset("key1", "val1", "key2", "val2")
```

### Hash Commands

```typescript
await redis.hset("user:1", "name", "Alice", "email", "alice@example.com")
await redis.hget("user:1", "name")            // "Alice"
await redis.hgetall("user:1")                 // ["name", "Alice", "email", "alice@example.com"]
await redis.hdel("user:1", "email")
await redis.hexists("user:1", "name")         // 0 | 1
await redis.hincrby("user:1", "visits", 1)
```

### List Commands

```typescript
await redis.lpush("queue", "item1", "item2")  // push to left
await redis.rpush("queue", "item3")            // push to right
await redis.lpop("queue")                      // pop from left
await redis.rpop("queue")                      // pop from right
await redis.lrange("queue", 0, -1)             // get all items
await redis.llen("queue")                      // length
```

### Set Commands

```typescript
await redis.sadd("tags", "js", "ts", "bun")
await redis.smembers("tags")                   // all members
await redis.sismember("tags", "bun")           // 0 | 1
await redis.srem("tags", "js")
await redis.scard("tags")                      // count
```

### Sorted Set Commands

```typescript
await redis.zadd("leaderboard", 100, "alice", 200, "bob")
await redis.zrange("leaderboard", 0, -1)               // by rank
await redis.zrangebyscore("leaderboard", 0, 150)        // by score
await redis.zscore("leaderboard", "alice")               // "100"
await redis.zrem("leaderboard", "alice")
await redis.zincrby("leaderboard", 50, "bob")
```

### Key Expiration

```typescript
await redis.expire("key", 3600)               // set TTL in seconds
await redis.pexpire("key", 5000)               // set TTL in milliseconds
await redis.ttl("key")                         // remaining TTL in seconds
await redis.pttl("key")                        // remaining TTL in ms
await redis.persist("key")                     // remove expiration
```

### Pub/Sub

```typescript
import { RedisClient } from "bun"

// Subscriber
const subscriber = new RedisClient("redis://localhost:6379")
await subscriber.connect()

await subscriber.subscribe("channel", (message, channel) => {
  console.log(`${channel}: ${message}`)
})

// Publisher (use a separate client)
const publisher = new RedisClient("redis://localhost:6379")
await publisher.connect()

await publisher.publish("channel", "Hello everyone!")

// Cleanup
subscriber.close()
publisher.close()
```

### Pipeline

Send multiple commands in a single round trip:

```typescript
const results = await redis.pipeline()
  .set("key1", "val1")
  .set("key2", "val2")
  .get("key1")
  .get("key2")
  .exec()
// results: array of command results
```

### Connection Management

```typescript
const client = new RedisClient("redis://localhost:6379")

// Explicit connect (optional -- commands auto-connect)
await client.connect()

// Close connection
client.close()

// Check if connected
client.isOpen  // boolean
```

### Common Patterns

```typescript
// Session store
async function getSession(sessionId: string) {
  const data = await redis.get(`session:${sessionId}`)
  return data ? JSON.parse(data) : null
}

async function setSession(sessionId: string, data: object, ttlSeconds = 3600) {
  await redis.set(`session:${sessionId}`, JSON.stringify(data), "EX", ttlSeconds)
}

// Rate limiter
async function checkRateLimit(ip: string, maxRequests = 100, windowSeconds = 60) {
  const key = `ratelimit:${ip}`
  const current = await redis.incr(key)
  if (current === 1) {
    await redis.expire(key, windowSeconds)
  }
  return current <= maxRequests
}

// Cache with TTL
async function cached<T>(key: string, ttl: number, fn: () => Promise<T>): Promise<T> {
  const hit = await redis.get(key)
  if (hit) return JSON.parse(hit)
  const result = await fn()
  await redis.set(key, JSON.stringify(result), "EX", ttl)
  return result
}
```
