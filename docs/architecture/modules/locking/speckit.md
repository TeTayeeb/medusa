# SpecKit — Locking Module

## Module Identity

| Attribute      | Value                                      |
|----------------|--------------------------------------------|
| Module Name    | `@medusajs/locking`                        |
| Version        | 2.15.4                                     |
| Module Key     | `locking`                                  |
| Constant       | `Modules.LOCKING`                          |
| Type           | Infrastructure / Concurrency Control       |
| Database Tables| None (in-process) / Redis keys / PG advisory locks |
| Event Emitter  | No                                         |
| Event Consumer | No                                         |

---

## Functional Specifications

### SPEC-LOCK-001: execute() Guarantee
**Description**: `execute(keys, job, options)` MUST acquire the lock(s), run `job()`, and release the lock(s) in `finally`. The lock MUST be released even if `job()` throws.  
**Acceptance**: `job()` throws → lock released; `release()` call count = `acquire()` call count.

### SPEC-LOCK-002: Mutual Exclusion
**Description**: Two concurrent calls to `execute()` with the same key MUST NOT run their jobs simultaneously. The second MUST wait until the first releases the lock.  
**Acceptance**: 50 concurrent `execute("same-key", incrementCounter)` calls → final counter value = 50 (no lost updates).

### SPEC-LOCK-003: TTL Auto-Release
**Description**: If a process holding a lock crashes or the job exceeds the TTL, the lock MUST auto-release after the specified `expire` duration.  
**Acceptance**: Lock acquired with `expire: 1000`; process hangs → lock released after 1000ms; other callers can acquire.

### SPEC-LOCK-004: Multi-Key Locking
**Description**: `acquire([keyA, keyB])` MUST acquire all keys atomically (or sequentially in lexicographic order). Partial acquisition is not permitted.  
**Acceptance**: Two processes acquiring `[keyA, keyB]` and `[keyB, keyA]` in lexicographic order → no deadlock.

### SPEC-LOCK-005: Retry on Contention
**Description**: When a lock is held, the acquiring call MUST retry up to `retryCount` times with `retryDelay` + random jitter between attempts before throwing `LockContentionError`.  
**Acceptance**: Lock held for 500ms; `retryCount: 3, retryDelay: 200` → call retries at ~200, 400, 600ms and either succeeds or throws after 3 failures.

### SPEC-LOCK-006: InProcess Provider Isolation
**Description**: The in-process provider MUST only prevent concurrency within a single Node.js process. It MUST NOT be advertised as distributed-safe.  
**Acceptance**: Documentation and error messages clearly state in-process limitation.

### SPEC-LOCK-007: Redis Provider Correctness
**Description**: The Redis provider MUST use a token-based locking mechanism (Redlock) to ensure only the lock holder can release the lock.  
**Acceptance**: Process B cannot release a lock held by Process A (different random token).

### SPEC-LOCK-008: PostgreSQL Advisory Lock Correctness
**Description**: The PostgreSQL provider MUST release advisory locks when the connection is returned to the pool. Lock release MUST happen in `finally`.  
**Acceptance**: DB connection returned to pool → advisory lock released; verified via `pg_locks` system view.

### SPEC-LOCK-009: acquire() and release() Direct Access
**Description**: `acquire()` and `release()` MUST be available as standalone methods for advanced use cases where `execute()` is insufficient.  
**Acceptance**: Manual acquire + release sequence works correctly without `execute()`.

---

## Non-Functional Specifications

### SPEC-LOCK-NFR-001: Lock Acquisition Latency
**Description**: Lock acquisition MUST complete in < 5ms p99 when no contention exists.  
**Target**: Redis provider: < 2ms; PostgreSQL provider: < 5ms; InProcess: < 0.1ms.

### SPEC-LOCK-NFR-002: Zero Infrastructure for Development
**Description**: The module MUST work in development using only the in-process provider with no Redis or PostgreSQL configuration beyond the existing DB.  
**Target**: `medusa start` with no locking config → in-process provider active; no errors.

---

## Module Service Contract

```typescript
interface ILockingModule {
  acquire(keys: string | string[], options?: { expire?: number }): Promise<void>
  release(keys: string | string[], options?: {}): Promise<boolean>
  execute<T>(keys: string | string[], job: () => Promise<T>, options?: { expire?: number }): Promise<T>
}
```

---

## Configuration Schema

```typescript
// In-process (default)
{ resolve: "@medusajs/locking" }

// Redis provider
{
  resolve: "@medusajs/locking-redis",
  options: {
    redisUrl: string
    retryCount?: number    // default: 3
    retryDelay?: number    // default: 200 (ms)
    retryJitter?: number   // default: 200 (ms)
  }
}

// PostgreSQL provider
{
  resolve: "@medusajs/locking-postgres",
  options: {
    retryCount?: number
    retryDelay?: number
  }
}
```

---

## Test Checklist

- [ ] `execute()` releases lock when job throws
- [ ] 50 concurrent increments via `execute()` → correct final value
- [ ] Lock TTL auto-releases after expire duration
- [ ] Multi-key: lexicographic ordering prevents deadlock
- [ ] Retry: `retryCount` exhausted → `LockContentionError` thrown
- [ ] Redis: Process B cannot release Process A's lock
- [ ] PG: advisory lock released on connection return
- [ ] In-process: works without Redis config

---

## Dependencies & Interfaces

| Dependency            | Interface Used             | Direction |
|-----------------------|----------------------------|-----------|
| Inventory module      | `execute()` for reservations | Outbound |
| Order workflows       | `execute()` for capture    | Outbound  |
| Cart workflows        | `execute()` for completion | Outbound  |
| Redis (optional)      | Redlock algorithm          | Outbound  |
| PostgreSQL (optional) | Advisory lock queries      | Outbound  |
