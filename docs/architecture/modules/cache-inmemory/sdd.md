# Software Design Document — Cache In-Memory

**Module:** `@medusajs/cache-inmemory`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `cache-inmemory` module implements Medusa's `ICacheService` interface using a bounded, LRU-evicting in-process Map. It provides fast, synchronous cache semantics with TTL support and zero external dependencies.

---

## 2. Goals and Non-Goals

### Goals
- Provide a working, zero-config cache for local development and testing.
- Implement the full `ICacheService` interface.
- Support per-entry TTL with lazy expiry checking.
- Bound memory usage via LRU eviction.

### Non-Goals
- Cross-process cache sharing.
- Persistence across restarts.
- Active TTL eviction (a background timer sweeping expired keys).

---

## 3. Architecture

### 3.1 Component Model

```
┌────────────────────────────────────────┐
│        Medusa Container (DI)           │
│                                        │
│  ┌────────────────────────────────┐   │
│  │    InMemoryCacheService        │   │
│  │  ────────────────────────────  │   │
│  │  + get(key): Promise<T|null>   │   │
│  │  + set(key, data, ttl?)        │   │
│  │  + invalidate(key)             │   │
│  │                                │   │
│  │  ┌──────────────────────────┐  │   │
│  │  │  LRU Map                 │  │   │
│  │  │  key → { value, expiresAt } │ │
│  │  └──────────────────────────┘  │   │
│  └────────────────────────────────┘   │
└────────────────────────────────────────┘
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `InMemoryCacheService` | Implements `ICacheService`; manages the LRU map |
| Module definition | Registers the service in the DI container with merged config |

### 3.3 LRU Map Implementation

The LRU map is implemented using a JavaScript `Map` (which preserves insertion order) combined with a size limit. When the limit is exceeded, the least-recently-inserted/accessed entry is evicted:

```typescript
class LRUMap<K, V> {
  private map: Map<K, V>
  private max: number

  set(key: K, value: V) {
    if (this.map.size >= this.max) {
      const oldest = this.map.keys().next().value
      this.map.delete(oldest)
    }
    this.map.set(key, value)
  }
}
```

### 3.4 TTL Handling

Each cache entry stores a computed `expiresAt` timestamp:

```typescript
interface CacheEntry<T> {
  data: T
  expiresAt: number | null    // null = never expire
}

// On get():
if (entry.expiresAt !== null && Date.now() > entry.expiresAt) {
  this.store.delete(key)
  return null
}
```

Expiry is checked lazily on `get()`; there is no background eviction timer.

---

## 4. Data Structures

```typescript
interface InMemoryCacheOptions {
  ttl?: number          // default TTL in seconds (default: 30)
  max?: number          // max number of entries (default: 1000)
}

// Internal storage
private store: Map<string, CacheEntry<unknown>>
```

---

## 5. Method Specifications

### `get<T>(key: string): Promise<T | null>`
1. Look up `key` in `store`.
2. If missing → return `null`.
3. If `expiresAt` is in the past → delete entry, return `null`.
4. Otherwise return `entry.data as T`.

### `set(key: string, data: unknown, ttl?: number): Promise<void>`
1. Compute `expiresAt = Date.now() + (ttl ?? this.options.ttl) * 1000`.
2. If `ttl === 0`, set `expiresAt = null`.
3. If `store.size >= max`, evict the first key.
4. `store.set(key, { data, expiresAt })`.

### `invalidate(key: string): Promise<void>`
1. `store.delete(key)`.

---

## 6. Error Handling

- All methods are wrapped in try/catch; exceptions are logged and re-thrown.
- `get()` never throws on a missing or expired key — it returns `null`.

---

## 7. Dependencies

| Dependency | Purpose |
|---|---|
| None (Node.js built-ins only) | `Map` for storage |
| `@medusajs/framework/types` | `ICacheService` interface |
| `@medusajs/framework/utils` | Module registration helpers |

---

## 8. Testing Strategy

- Unit tests cover: set/get round-trip, TTL expiry, LRU eviction at capacity, invalidate.
- Integration tests verify cache is populated and cleared by Medusa core logic (e.g., store config caching).
