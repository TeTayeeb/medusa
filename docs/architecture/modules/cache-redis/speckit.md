# SpecKit — Cache Redis

**Module:** `@medusajs/cache-redis`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/cache-redis` |
| DI key | `Modules.CACHE` |
| Interface | `ICacheService` |
| Category | Infrastructure / Cache |
| Default for | Production multi-process deployments |
| External dependency | Redis ≥ 5.0 |

---

## 2. Functional Specifications

### SPEC-CRD-001 — Set stores value with TTL
**Given** the module is connected to Redis
**When** `cache.set("k", { foo: "bar" }, 30)` is called
**Then** Redis contains key `"k"` with value `JSON.stringify({ foo: "bar" })`.
**And** the Redis key TTL is approximately 30 seconds (`TTL k` returns a value between 28 and 30).

### SPEC-CRD-002 — Get returns deserialised value
**Given** Redis contains key `"k"` with value `'{"foo":"bar"}'`
**When** `cache.get<{ foo: string }>("k")` is called
**Then** the result equals `{ foo: "bar" }` (parsed from JSON).

### SPEC-CRD-003 — Get returns null for missing key
**Given** key `"missing"` does not exist in Redis
**When** `cache.get("missing")` is called
**Then** the result is `null` and no exception is thrown.

### SPEC-CRD-004 — Get returns null after Redis TTL expiry
**Given** `cache.set("k", value, 1)` is called with TTL = 1 second
**When** `cache.get("k")` is called 1100 ms later
**Then** the result is `null` (Redis has evicted the key).

### SPEC-CRD-005 — Invalidate removes key from Redis
**Given** `cache.set("k", value)` is called
**When** `cache.invalidate("k")` is called
**Then** `GET k` on Redis returns `nil`.
**And** `cache.get("k")` returns `null`.

### SPEC-CRD-006 — Cross-process consistency
**Given** Process A calls `cache.set("k", v)`
**When** Process B calls `cache.get("k")`
**Then** Process B receives `v`.

### SPEC-CRD-007 — Cross-process invalidation
**Given** Process A calls `cache.set("k", v)`
**When** Process A calls `cache.invalidate("k")`
**Then** Process B's `cache.get("k")` returns `null`.

### SPEC-CRD-008 — TTL = 0 sets key with no expiry
**Given** `cache.set("k", value, 0)` is called
**Then** `TTL k` on Redis returns `-1` (no expiry).

### SPEC-CRD-009 — Deserialisation failure returns null
**Given** Redis contains a malformed JSON value for key `"k"`
**When** `cache.get("k")` is called
**Then** the result is `null` and the error is logged.
**And** no exception propagates to the caller.

### SPEC-CRD-010 — Interface compatibility
The module must satisfy `ICacheService` exactly. All tests written against `cache-inmemory` must pass unchanged against `cache-redis`.

---

## 3. Non-Functional Specifications

### SPEC-CRD-NF-001 — TLS support
The module must support `redss://` URL scheme for encrypted Redis connections.

### SPEC-CRD-NF-002 — Automatic reconnection
On Redis disconnect, `ioredis` must retry automatically; subsequent `get/set/invalidate` calls must succeed after reconnection without process restart.

### SPEC-CRD-NF-003 — `redisOptions` passthrough
All fields of `options.redisOptions` must be forwarded to the `ioredis` constructor unchanged.

---

## 4. Configuration Specification

```typescript
interface CacheRedisOptions {
  redisUrl: string               // required
  ttl?: number                   // default: 30 seconds
  redisOptions?: RedisOptions    // ioredis constructor options
}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-CRD-001 | Integration (real Redis) | Required |
| SPEC-CRD-002 | Integration | Required |
| SPEC-CRD-003 | Integration | Required |
| SPEC-CRD-004 | Integration | Required |
| SPEC-CRD-005 | Integration | Required |
| SPEC-CRD-006 | Integration (2-process) | Required |
| SPEC-CRD-007 | Integration (2-process) | Required |
| SPEC-CRD-008 | Integration | Required |
| SPEC-CRD-009 | Unit (mock Redis returning garbage) | Required |
| SPEC-CRD-010 | Contract tests | Required |
| SPEC-CRD-NF-001 | Integration (TLS Redis) | Optional |
| SPEC-CRD-NF-002 | Integration | Required |
| SPEC-CRD-NF-003 | Unit | Required |

---

## 6. Out of Scope

- Cache stampede protection
- Pattern-based invalidation (`DEL medusa:*`)
- Cache warming / pre-population
- Cross-datacenter replication
- Monitoring / metrics export
