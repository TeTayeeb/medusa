# arc42 Architecture Documentation — Cache Redis

**Module:** `@medusajs/cache-redis`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The cache-redis module provides a distributed, Redis-backed cache shared across all Medusa processes. It must implement `ICacheService`, leverage Redis native TTL, serialise complex objects, and support TLS for managed Redis services.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Consistency** | All API pods read the same cached value after any one of them writes it |
| 2 | **Correctness** | TTL expiry is accurate and handled by Redis, not the client |
| 3 | **Resilience** | Transient Redis disconnects are handled gracefully |

---

## 2. Architecture Constraints

- Must use `ioredis` (already a Medusa dependency).
- Must not implement its own TTL scheduler — delegate to Redis `EXPIRE`.
- Must be fully interface-compatible with `cache-inmemory`.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────────────────┐
│ External: Redis Server                                          │
│   Keys: medusa:store:config:default  (TTL: 30s)                │
│         medusa:tax:rate:region_01HX  (TTL: 30s)                │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ ioredis
              ┌────────────────────┴────────────────────┐
              │ API Process A        API Process B       │
              │ cache.get/set/del    cache.get/set/del   │
              └─────────────────────────────────────────┘
```

---

## 4. Solution Strategy

All cache operations map directly to Redis commands: `GET` → `get()`, `SET ... EX` → `set()`, `DEL` → `invalidate()`. JSON serialisation handles complex objects. `ioredis` provides connection pooling, automatic reconnection, and TLS support. The module is a thin adapter between `ICacheService` and `ioredis`.

---

## 5. Building Blocks

```
cache-redis
  ├── RedisCacheService     (ICacheService impl)
  ├── RedisConnection       (ioredis lifecycle wrapper)
  └── Serialiser            (JSON.stringify / JSON.parse)
```

### Operation Mapping

| ICacheService | Redis Command |
|---|---|
| `get(key)` | `GET key` |
| `set(key, v, ttl)` | `SET key <json> EX ttl` |
| `set(key, v, 0)` | `SET key <json>` (no expiry) |
| `invalidate(key)` | `DEL key` |

---

## 6. Runtime View

### Scenario: Cross-Process Cache Consistency

```
API Pod A                     Redis                  API Pod B
  │                             │                       │
  ├── cache.set("store", v) ──► SET store <json> EX 30  │
  │                             │                       │
  │                             │ ◄── GET store ────────┤
  │                             │ ──► <json> ──────────►│ (same value)
  │                             │                       │
  ├── cache.invalidate("store") DEL store               │
  │                             │                       │
  │                             │ ◄── GET store ────────┤
  │                             │ ──► null ────────────►│ (cache miss)
```

---

## 7. Deployment View

```
┌──────── Production Deployment ────────────────────────────────┐
│                                                               │
│  ┌─────────────────┐    ┌──────────────────────────────────┐  │
│  │  Medusa Pods ×N  │    │  Redis (Sentinel / Cluster)      │  │
│  │  RedisCacheService│──►│  Keys: medusa:*                  │  │
│  └─────────────────┘    └──────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### Connection Resilience
`ioredis` automatically reconnects on disconnect. During reconnection, `get()` / `set()` calls are queued and replayed. `commandTimeout` can be set to fail fast if Redis is unresponsive.

### JSON Serialisation Boundary
`Date` objects round-trip as ISO strings. Code that reads cached `Date` fields must re-hydrate them: `new Date(cached.createdAt)`.

### Key Namespace
All keys are prefixed `medusa:` by consuming modules. The cache module itself imposes no key restrictions.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Delegate TTL to Redis EXPIRE | Redis handles expiry atomically; no client-side scheduler drift |
| AD-2 | JSON serialisation | Universal; works with any POJO; no binary protocol needed |
| AD-3 | `ioredis` over `redis` package | Better TypeScript support, cluster/sentinel, stream commands |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Consistency | Integration test: two processes write/read same key |
| TTL accuracy | Integration test: key absent after TTL + 1s |
| Resilience | Unit test: `ioredis` error event → method rejects, caller falls through |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Redis single point of failure | High | Use Redis Sentinel or Cluster |
| Cache stampede on cold start | Medium | Consider probabilistic early expiry or request coalescing |
| Large serialised values | Low-Medium | Monitor Redis memory; set MAXMEMORY policy |
