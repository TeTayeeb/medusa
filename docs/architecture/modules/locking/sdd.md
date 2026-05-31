# Software Design Document — Locking Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa Locking module (v2.15.4). It covers the provider interface, the three built-in provider implementations, lock acquisition and release algorithms, TTL mechanics, retry strategies, and integration patterns with critical business workflows.

## 2. Architecture Overview

```
Business Workflow (e.g., complete-cart)
  ↓
ILockingModule.execute("cart:complete:crt_01", callback, { expire: 5000 })
  ↓
Active Provider (InProcess | Redis | PostgreSQL)
  ├─ acquire(key, ttl)
  │    ├─ success → run callback → release(key) → return result
  │    └─ contention → retry (backoff) → timeout → throw LockError
  └─ release(key)
```

## 3. Provider Interface

```typescript
interface ILockingProvider {
  acquire(keys: string[], options?: { expire?: number }): Promise<void>
  release(keys: string[], options?: {}): Promise<boolean>
  execute<T>(keys: string[], job: () => Promise<T>, options?: { expire?: number }): Promise<T>
}
```

The `execute` method is the preferred API. It guarantees release even if the job throws an exception.

## 4. InProcess Provider

**Use case**: Single-process development environments.

**Implementation**: Uses a `Map<string, Promise<void>>` as a mutex registry. Each lock is represented by a chain of promises; acquisition appends to the chain.

```typescript
private locks = new Map<string, { resolve: () => void; timer?: NodeJS.Timeout }[]>()

async acquire(keys: string[], { expire = 30000 }) {
  for (const key of keys) {
    await new Promise<void>((resolve, reject) => {
      const entry = { resolve, timer: setTimeout(() => reject(new LockTimeoutError(key)), expire) }
      const queue = this.locks.get(key) ?? []
      this.locks.set(key, [...queue, entry])
      if (queue.length === 0) resolve() // first in queue, acquire immediately
    })
  }
}
```

**Limitation**: Not safe across multiple Node.js processes or workers.

## 5. Redis Provider

**Use case**: Production multi-process deployments.

**Implementation**: Uses the Redlock algorithm (Martin Kleppmann) via the `redlock` npm package.

Key aspects:
- Lock key: `medusa:lock:{key}` in Redis.
- Value: cryptographically random token (prevents accidental release by another process).
- TTL: set on the Redis key; auto-expires if the process crashes.
- Retry: configurable count and jitter-based delay via `retryCount` and `retryDelay`.

```typescript
async acquire(keys: string[], { expire = 5000 }) {
  const redisKeys = keys.map(k => `medusa:lock:${k}`)
  this.activeLocks.set(keys.join(","), await this.redlock.acquire(redisKeys, expire))
}

async release(keys: string[]) {
  const lock = this.activeLocks.get(keys.join(","))
  if (lock) {
    await this.redlock.release(lock)
    this.activeLocks.delete(keys.join(","))
    return true
  }
  return false
}
```

## 6. PostgreSQL Provider

**Use case**: Production deployments without a Redis dependency.

**Implementation**: Uses PostgreSQL advisory locks via `pg_try_advisory_lock(key_hash)`.

```typescript
async acquire(keys: string[], { expire = 30000 }) {
  for (const key of keys) {
    const hash = hashKeyToInt64(key) // deterministic CRC32/bigint
    const result = await this.db.query(
      "SELECT pg_try_advisory_lock($1)", [hash]
    )
    if (!result.rows[0].pg_try_advisory_lock) {
      throw new LockContentionError(key)
    }
  }
}
```

Advisory locks are session-scoped: released automatically on connection close, preventing deadlocks from crashed workers.

## 7. execute() Implementation Pattern

All providers implement `execute` using this pattern:

```typescript
async execute<T>(keys: string[], job: () => Promise<T>, options?) {
  await this.acquire(keys, options)
  try {
    return await job()
  } finally {
    await this.release(keys)
  }
}
```

The `finally` block ensures the lock is always released, even if `job()` throws.

## 8. TTL and Deadlock Prevention

| Provider    | TTL Mechanism                        | Deadlock Prevention                       |
|-------------|--------------------------------------|-------------------------------------------|
| InProcess   | `setTimeout` → reject                | Process restart clears all locks          |
| Redis       | Redis key TTL (SET ... PX)           | Auto-expire after TTL even if not released|
| PostgreSQL  | Timeout via `lock_timeout` GUC       | Session end releases advisory lock        |

## 9. Lock Key Conventions

Lock keys should be scoped to the specific resource being locked:

```
{domain}:{operation}:{resource_id}
```

Examples:
- `inventory:reserve:variant_01HXXXXX`
- `order:capture:ord_01HXXXXX`
- `cart:complete:cart_01HXXXXX`

Using resource IDs (not types alone) ensures maximum concurrency: two different orders can be captured simultaneously.

## 10. Multi-Key Locking

For operations spanning multiple resources (e.g., multi-item reservation):

```typescript
await lockingModule.execute(
  ["inventory:reserve:var_A", "inventory:reserve:var_B"],
  async () => { /* reserve both */ },
  { expire: 10000 }
)
```

Keys are acquired in lexicographic order to prevent deadlocks from two concurrent processes acquiring the same keys in different orders.

## 11. Error Types

| Error Class          | Cause                                     | Retry?  |
|----------------------|-------------------------------------------|---------|
| `LockTimeoutError`   | TTL expired before lock was acquired      | No      |
| `LockContentionError`| All retries exhausted under contention    | Manual  |
| `LockReleaseError`   | Lock token mismatch (stale lock)          | No      |

## 12. Configuration Options

| Option         | Type   | Default | Description                                  |
|----------------|--------|---------|----------------------------------------------|
| `redisUrl`     | string | —       | Redis connection URL (Redis provider)        |
| `retryCount`   | number | `3`     | Max acquisition retries                      |
| `retryDelay`   | number | `200`   | Base delay between retries (ms)              |
| `retryJitter`  | number | `200`   | Random jitter added to retry delay (ms)      |
