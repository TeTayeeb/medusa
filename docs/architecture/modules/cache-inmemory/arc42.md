# arc42 Architecture Documentation — Cache In-Memory

**Module:** `@medusajs/cache-inmemory`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The cache-inmemory module provides a fast, zero-dependency cache for development and testing. It must implement `ICacheService`, support TTL, bound memory usage via LRU eviction, and require no configuration or external services.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Simplicity** | Works out of the box with no config |
| 2 | **Speed** | Sub-millisecond cache reads (no network) |
| 3 | **Predictability** | TTL expiry and LRU eviction behave as documented |

---

## 2. Architecture Constraints

- Must not introduce npm dependencies beyond `@medusajs/framework`.
- Must be fully interface-compatible with `cache-redis`.
- Must not use `setInterval` or background timers (keep the process clean).

---

## 3. System Scope and Context

```
┌──────────────────────────────────────────────────────┐
│                  Medusa Application                   │
│                                                      │
│  Store config lookup ─────► cache.get("store:cfg")   │
│  Tax rate lookup     ─────► cache.get("tax:rate:x")  │
│                                                      │
│       ┌──────────────────────────────────┐          │
│       │   cache-inmemory                 │          │
│       │   (LRU Map — in-process only)    │          │
│       └──────────────────────────────────┘          │
│                                                      │
│  [no external systems]                               │
└──────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

A bounded `Map<string, { data, expiresAt }>` serves as the cache store. Expiry is checked lazily on `get()`. LRU eviction removes the oldest inserted key when the map exceeds its `max` capacity. All async method signatures return immediately-resolving Promises to maintain interface compatibility with `cache-redis`.

---

## 5. Building Blocks

```
cache-inmemory
  └── InMemoryCacheService  (ICacheService impl)
        ├── LRUMap           (bounded Map with insertion-order tracking)
        └── TTL checker      (lazy expiry on get)
```

### Method Summary

| Method | Complexity | Notes |
|---|---|---|
| `get(key)` | O(1) | Includes TTL check; lazy delete on miss |
| `set(key, v, ttl?)` | O(1) amortised | LRU eviction if at capacity |
| `invalidate(key)` | O(1) | `Map.delete` |

---

## 6. Runtime View

### Scenario: Store Config Cache Hit

```
HTTP Request: GET /store
  │
  ▼
StoreModule.retrieve()
  │── cache.get("store:config:default")
  │     └── Map.get → { data, expiresAt }
  │           └── expiresAt > now → return data  ← CACHE HIT
  │
  └──► return store config (no DB query)
```

### Scenario: Cache Miss + Populate

```
cache.get("store:config:default") → null (expired or absent)
  │
  └──► DB query: SELECT * FROM store WHERE id = 'default'
         │
         └──► cache.set("store:config:default", result, 30)
```

---

## 7. Deployment View

No dedicated deployment. Runs embedded in the Medusa Node.js process. Cache state is per-process and ephemeral.

```
Node.js Process
┌──────────────────────────────────────┐
│  Medusa Server                       │
│  InMemoryCacheService                │
│    LRUMap (max=1000, in heap)        │
└──────────────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### Memory Management
The `max` option caps heap usage. Default of 1000 entries is safe for typical Medusa installations. Large stores with many tax rates or product categories should tune this upward.

### Serialisation
No serialisation is performed; values are stored as-is in JavaScript heap. This means objects are stored by reference — callers should not mutate returned values.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Lazy TTL eviction | Avoids timer overhead; acceptable for cache use patterns |
| AD-2 | Map-based LRU | Simple, fast, no dependencies |
| AD-3 | Promise-wrapped returns | Interface parity with `cache-redis` |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Zero config | Module registers with no required options |
| Correct TTL | `get()` returns null after TTL seconds; unit tested |
| Bounded memory | LRU eviction at `max` capacity; unit tested |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Stale data across processes | Low (dev only) | By design; use `cache-redis` in multi-process |
| Memory leak if `max` set too high | Low | Document and default to 1000 |
| Object mutation via shared reference | Low | Document; callers should treat returned values as immutable |
