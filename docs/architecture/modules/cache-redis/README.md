# Cache Redis Module

## Overview

The `cache-redis` module provides Medusa's distributed, Redis-backed cache service. It implements the `ICacheService` interface using `ioredis` for Redis connectivity, and is the recommended cache module for production deployments where multiple API and worker processes must share a consistent view of cached data.

## Purpose

When Medusa runs as multiple horizontally scaled processes, an in-memory cache per process leads to stale reads and redundant database queries. The Redis cache solves this by providing a single shared store: when one API process invalidates a tax-rate cache entry, all other processes immediately see the invalidation on their next read.

## Key Features

- **Distributed** — all Medusa processes share a single Redis instance; cache coherence is maintained automatically.
- **TTL via Redis EXPIRE** — TTL is delegated to Redis's native key expiry, meaning eviction is accurate and automatic without any in-process scheduler.
- **JSON serialisation** — complex objects (nested DTOs, arrays) are serialised to JSON on write and deserialised on read transparently.
- **Configurable defaults** — a module-level default TTL can be set; individual `set()` calls override it per key.
- **TLS support** — `redisUrl` with `rediss://` scheme enables TLS-encrypted connections to managed Redis services (ElastiCache, Upstash, etc.).
- **Drop-in replacement** — satisfies the same `ICacheService` interface as `cache-inmemory`.

## Interface

```typescript
interface ICacheService {
  get<T>(key: string): Promise<T | null>
  set(key: string, data: unknown, ttl?: number): Promise<void>
  invalidate(key: string): Promise<void>
}
```

## Configuration

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/cache-redis",
      key: Modules.CACHE,
      options: {
        redisUrl: process.env.REDIS_URL,  // required — e.g. redis://localhost:6379
        ttl: 30,                           // default TTL in seconds
        redisOptions: {                    // passed directly to ioredis
          tls: {},
          maxRetriesPerRequest: 3,
        },
      },
    },
  ],
})
```

## Key Naming Convention

Medusa builds cache keys using a colon-delimited namespace pattern:

```
medusa:<module>:<resource>:<id>
```

For example: `medusa:store:config:default`, `medusa:tax:rate:region_01HX...`

## Limitations

| Constraint | Detail |
|---|---|
| **Redis required** | Needs a running Redis ≥ 5.0 instance. |
| **Network latency** | Every cache hit incurs a network round-trip (typically < 1 ms on localhost / same AZ). |
| **Serialisation cost** | Large objects with deep nesting may incur non-trivial JSON serialise/deserialise overhead. |
| **No built-in cache stampede protection** | Concurrent misses may all reach the database simultaneously. |

## When to Use

| Scenario | Recommendation |
|---|---|
| Multi-process / Kubernetes deployment | ✅ Required |
| Shared cache across API and worker processes | ✅ Use this |
| Managed Redis (ElastiCache, Upstash) | ✅ Fully supported |
| Local development / tests | ❌ Use `cache-inmemory` |

## Related Modules

- [`cache-inmemory`](../cache-inmemory/README.md) — development alternative.
