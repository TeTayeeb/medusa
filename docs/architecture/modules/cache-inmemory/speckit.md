# SpecKit — Cache In-Memory

**Module:** `@medusajs/cache-inmemory`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/cache-inmemory` |
| DI key | `Modules.CACHE` |
| Interface | `ICacheService` |
| Category | Infrastructure / Cache |
| Default for | Development and testing |

---

## 2. Functional Specifications

### SPEC-CIM-001 — Set and Get round-trip
**Given** `cache.set("k", { foo: "bar" })` is called
**When** `cache.get("k")` is called before TTL expiry
**Then** the result equals `{ foo: "bar" }`.

### SPEC-CIM-002 — TTL expiry
**Given** `cache.set("k", value, 1)` is called with TTL of 1 second
**When** `cache.get("k")` is called after 1001 ms
**Then** the result is `null`.

### SPEC-CIM-003 — TTL = 0 means never expire
**Given** `cache.set("k", value, 0)` is called
**When** `cache.get("k")` is called at any time during the process lifetime
**Then** the result equals `value` (no expiry).

### SPEC-CIM-004 — Get missing key
**Given** key `"missing"` has never been set
**When** `cache.get("missing")` is called
**Then** the result is `null` and no exception is thrown.

### SPEC-CIM-005 — Invalidate removes entry
**Given** `cache.set("k", value)` is called
**When** `cache.invalidate("k")` is called
**Then** `cache.get("k")` returns `null`.

### SPEC-CIM-006 — LRU eviction at capacity
**Given** `options.max = 3` and entries `"a"`, `"b"`, `"c"` are set
**When** entry `"d"` is set
**Then** entry `"a"` (the first/oldest) is evicted.
**And** `cache.get("a")` returns `null`.
**And** `cache.get("d")` returns the value set for `"d"`.

### SPEC-CIM-007 — Overwrite existing key
**Given** `cache.set("k", "v1")` is called
**When** `cache.set("k", "v2")` is called
**Then** `cache.get("k")` returns `"v2"`.

### SPEC-CIM-008 — Default TTL from options
**Given** `options.ttl = 10` (module-level default)
**And** `cache.set("k", value)` is called without an explicit TTL argument
**Then** the entry expires after 10 seconds.

---

## 3. Non-Functional Specifications

### SPEC-CIM-NF-001 — Zero dependencies
The module must have no runtime npm dependencies outside `@medusajs/framework`.

### SPEC-CIM-NF-002 — Interface compatibility
The module must satisfy `ICacheService` exactly. All tests written against `cache-redis` must pass unchanged against `cache-inmemory`.

### SPEC-CIM-NF-003 — Lazy TTL eviction only
No `setInterval` or background timer must be created by the module.

---

## 4. Configuration Specification

```typescript
interface InMemoryCacheOptions {
  ttl?: number    // default TTL in seconds (default: 30; 0 = never expire)
  max?: number    // max number of entries (default: 1000)
}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-CIM-001 | Unit | Required |
| SPEC-CIM-002 | Unit (with fake timers) | Required |
| SPEC-CIM-003 | Unit | Required |
| SPEC-CIM-004 | Unit | Required |
| SPEC-CIM-005 | Unit | Required |
| SPEC-CIM-006 | Unit | Required |
| SPEC-CIM-007 | Unit | Required |
| SPEC-CIM-008 | Unit | Required |
| SPEC-CIM-NF-001 | Static analysis | Required |
| SPEC-CIM-NF-002 | Contract tests | Required |
| SPEC-CIM-NF-003 | Unit (spy on global setInterval) | Required |

---

## 6. Out of Scope

- Cross-process cache sharing
- Persistence across restarts
- Active background eviction sweeps
- Cache stampede protection
- Pattern-based invalidation (e.g., `invalidate("medusa:tax:*")`)
