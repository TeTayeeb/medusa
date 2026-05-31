# @medusajs/caching-redis

Redis-backed caching provider for Medusa v2. Implements `ICacheService` and registers under `Modules.CACHING` with identifier `RedisCachingProvider`.

Provides a high-performance, tag-based cache with automatic compression, TTL support, and graceful degradation on Redis connection failures.

## Installation

```bash
npm install @medusajs/caching-redis
```

## Configuration

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/cache-redis",
      options: {
        redisUrl: process.env.CACHE_REDIS_URL,   // Redis connection URL
        ttl: 3600,                                // Default TTL in seconds (optional)
        prefix: "mc:",                            // Key prefix (optional)
      },
    },
  ],
})
```

### Options reference

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `redisUrl` | `string` | ✅ | — | Redis connection URL (e.g. `redis://localhost:6379`) |
| `ttl` | `number` | — | `3600` | Default cache TTL in seconds (1 hour) |
| `prefix` | `string` | — | `"mc:"` | Key namespace prefix to avoid collisions |
| `connectTimeout` | `number` | — | `10000` | Connection timeout in ms |
| `commandTimeout` | `number` | — | `5000` | Command timeout in ms |
| `maxRetriesPerRequest` | `number` | — | `3` | Max retries per failed Redis command |

Additional ioredis options (except `redisUrl`) can be passed and are forwarded directly to the ioredis constructor.

## Core operations

### `get({ key, tags? })`

```ts
// Single key lookup
const value = await cache.get({ key: "product:prod_123" })

// Tag-based lookup (returns all values associated with a tag)
const values = await cache.get({ tags: ["product-list", "category:electronics"] })
```

- Returns `null` (single key) or `[]` (tag query) on cache miss or Redis error.
- Data is decompressed before return.

### `set({ key, data, ttl?, tags?, options? })`

```ts
await cache.set({
  key: "product:prod_123",
  data: { id: "prod_123", title: "Widget" },
  ttl: 1800,                       // Override default TTL
  tags: ["product-list"],          // Associate with invalidation tags
  options: { autoInvalidate: true }, // Auto-invalidate on related tag clear
})
```

- Data is JSON-serialized and compressed before storage.
- Tags are interned (referenced by numeric IDs) to minimise Redis memory usage.
- Uses pipeline for atomic multi-command execution.

### `invalidate(key | tag)`

```ts
// Invalidate single key
await cache.invalidate("product:prod_123")

// Invalidate all entries tagged with a tag
await cache.invalidate("product-list")
```

Invalidation:
1. Deletes the cache key and its associated tags key.
2. Decrements reference counts for all associated tags.
3. When a tag key is directly invalidated: scans all members of the tag set, deletes each cached entry (checking `autoInvalidate` option per entry).

## Tag-based invalidation

Tags allow grouped invalidation of related cache entries:

```ts
// Store multiple entries under the same tag
await cache.set({ key: "product:1", data: {...}, tags: ["products"] })
await cache.set({ key: "product:2", data: {...}, tags: ["products"] })

// Invalidate all products at once
await cache.invalidate("products")
// → both "product:1" and "product:2" are removed
```

## Graceful degradation

Redis connection errors during `get` and `set` log warnings and return `null` / `[]` respectively — they do **not** throw. This ensures application operations continue unimpeded when Redis is temporarily unavailable.

## Connection lifecycle

The provider uses a connection loader (`loaders/connection.js`) that:
1. Creates an ioredis client with `lazyConnect: true`
2. Tests the connection with `PING` at startup
3. Registers `redisClient`, `prefix`, and `logger` in the Medusa DI container

Connection failures during PING are logged as warnings but do not prevent startup.

## Environment variables

```dotenv
CACHE_REDIS_URL=redis://localhost:6379
```
