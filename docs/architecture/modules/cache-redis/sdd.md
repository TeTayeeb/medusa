# Software Design Document — Cache Redis

**Module:** `@medusajs/cache-redis`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `cache-redis` module implements `ICacheService` using Redis as the backing store via `ioredis`. It provides distributed, TTL-aware caching shared across all Medusa processes, and is the recommended cache for production deployments.

---

## 2. Goals and Non-Goals

### Goals
- Provide a cache consistent across all API and worker processes.
- Delegate TTL eviction entirely to Redis's native `EXPIRE` mechanism.
- Serialise and deserialise complex objects transparently.
- Support TLS connections for managed Redis services.

### Non-Goals
- Cache-aside pattern implementation (callers are responsible for the miss/populate logic).
- Built-in cache stampede protection.
- Cross-datacenter replication.

---

## 3. Architecture

### 3.1 Component Model

```
Process A (API)               Redis                  Process B (Worker)
──────────────────            ──────────────────     ──────────────────
cache.set("k", v, 30) ──────► SET k <json> EX 30
cache.get("k")        ──────► GET k             ──►  cache.get("k")
                              ← <json>               ← same <json>
cache.invalidate("k") ──────► DEL k             ──►  cache.get("k") → null
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `RedisCacheService` | Implements `ICacheService`; wraps ioredis GET/SET/DEL |
| `RedisConnection` | Manages `ioredis` client, handles reconnect |
| Module definition | Registers service with merged config |

### 3.3 Serialisation Strategy

All values are serialised to JSON before storage and deserialised on retrieval:

```typescript
// set
const serialised = JSON.stringify(data)
await this.redis.set(key, serialised, "EX", ttl)

// get
const raw = await this.redis.get(key)
if (!raw) return null
return JSON.parse(raw) as T
```

**Limitations of JSON serialisation:**
- `Date` objects are serialised as ISO strings; callers must re-hydrate if needed.
- `undefined` values are dropped by `JSON.stringify`.
- Circular references throw; complex objects must be plain POJOs.

### 3.4 TTL Delegation

Rather than maintaining an in-process expiry timer, `SET key value EX <seconds>` is used. Redis atomically associates the TTL; no client-side bookkeeping is needed. `invalidate()` uses `DEL` which also removes any associated TTL.

---

## 4. Data Structures

```typescript
interface CacheRedisOptions {
  redisUrl: string               // required
  ttl?: number                   // default TTL in seconds (default: 30)
  redisOptions?: RedisOptions    // ioredis passthrough
}
```

---

## 5. Method Specifications

### `get<T>(key: string): Promise<T | null>`
1. `GET key` via ioredis.
2. If `null` → return `null` (Redis handles expiry automatically).
3. `JSON.parse(raw)` → return `T`.
4. On `JSON.parse` error → log error, return `null`.

### `set(key: string, data: unknown, ttl?: number): Promise<void>`
1. `serialised = JSON.stringify(data)`.
2. `effectiveTtl = ttl ?? this.options.ttl`.
3. If `effectiveTtl > 0`: `SET key serialised EX effectiveTtl`.
4. If `effectiveTtl === 0`: `SET key serialised` (no expiry).

### `invalidate(key: string): Promise<void>`
1. `DEL key`.

---

## 6. Error Handling

- **Connection errors** — `ioredis` emits `error` events; `RedisCacheService` logs them and allows the client to reconnect.
- **Deserialisation errors** — logged; `null` returned so callers fall through to the database.
- **Timeout** — `ioredis` `commandTimeout` option can be set via `redisOptions`.

---

## 7. Dependencies

| Dependency | Purpose |
|---|---|
| `ioredis` | Redis client (SET, GET, DEL, EXPIRE) |
| `@medusajs/framework/types` | `ICacheService` interface |
| `@medusajs/framework/utils` | Module registration helpers |

---

## 8. Testing Strategy

- Unit tests mock `ioredis` and verify: SET with correct EX, GET returning parsed value, DEL on invalidate, JSON round-trip correctness.
- Integration tests use a real Redis instance to verify: TTL expiry (with a short TTL and a sleep), cross-call consistency, handling of non-existent keys.
- A smoke test verifies that `cache-redis` and `cache-inmemory` produce identical observable behaviour against the `ICacheService` contract.
