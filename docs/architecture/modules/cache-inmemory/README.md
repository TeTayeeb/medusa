# Cache In-Memory Module

## Overview

The `cache-inmemory` module provides Medusa's in-process, LRU-backed cache service. It implements the `ICacheService` interface using a simple in-memory Map with TTL eviction, and requires no external infrastructure. It is the default cache for development and test environments.

## Purpose

Medusa's query layer and several core modules use a shared cache to reduce repeated database round-trips (e.g., store configuration lookups, tax rate resolutions). The in-memory cache satisfies this contract without any network overhead, making it ideal for single-process deployments and automated tests.

## Key Features

- **Zero dependencies** — pure JavaScript Map; no Redis, no Memcached.
- **LRU eviction** — least-recently-used entries are evicted when the cache reaches its size limit.
- **TTL support** — each entry can carry an expiry time; expired entries are excluded on read.
- **Synchronous-friendly** — `get()` resolves immediately without network I/O.
- **Drop-in replacement** — satisfies the same `ICacheService` interface as `cache-redis`.

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
      resolve: "@medusajs/cache-inmemory",
      key: Modules.CACHE,
      options: {
        ttl: 30,       // default TTL in seconds (0 = never expire)
        max: 1000,     // maximum number of entries
      },
    },
  ],
})
```

The module is the default when no cache module is explicitly configured.

## TTL Behaviour

- If `ttl` is `0` or omitted, entries never expire (bounded only by `max`).
- Each `set()` call optionally overrides the module-level default TTL for that specific key.
- Expired entries are checked lazily on `get()` and immediately discarded.

## Limitations

| Constraint | Detail |
|---|---|
| **Not shared** | Each process has its own isolated cache; values set by one process are invisible to others. |
| **No persistence** | Cache is wiped on process restart. |
| **Memory pressure** | Large caches consume heap memory; tune `max` appropriately. |
| **No pub/sub invalidation** | There is no mechanism to invalidate a key across all processes simultaneously. |

## When to Use

| Scenario | Recommendation |
|---|---|
| Local development | ✅ Default choice |
| Unit / integration tests | ✅ Ideal |
| Single-process deployments | ✅ Acceptable |
| Multi-process / scaled deployments | ❌ Use `cache-redis` |
| Consistent cache across workers | ❌ Use `cache-redis` |

## Related Modules

- [`cache-redis`](../cache-redis/README.md) — distributed Redis-backed alternative for production.
