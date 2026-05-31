# arc42 Architecture Document — Locking Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Locking module must prevent race conditions in concurrent business operations (inventory reservation, order capture, cart completion), support development-time use without external infrastructure, and scale to production multi-process deployments with Redis or PostgreSQL backends.

### 1.2 Quality Goals

| Priority | Quality Goal   | Scenario                                                            |
|----------|----------------|---------------------------------------------------------------------|
| 1        | Correctness    | Two concurrent checkout requests for the last item in stock: only one succeeds |
| 2        | Safety         | A process crash never leaves a lock permanently acquired            |
| 3        | Simplicity     | Developer writes two lines to wrap a critical section               |
| 4        | Replaceability | Switch from in-process to Redis provider via config change          |

### 1.3 Stakeholders

| Role           | Expectation                                                   |
|----------------|---------------------------------------------------------------|
| Platform devops| Production-grade distributed locking without custom code      |
| Backend dev    | Simple `execute(key, job)` API; no manual acquire/release     |
| SRE            | Lock TTL prevents permanent deadlocks; observable via metrics |

---

## 2. Architecture Constraints

- The `execute()` method must always release the lock, even if the job throws.
- Lock TTL must be configurable per call site; no global-only default.
- The module must not require Redis for local development.

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────┐
│               Business Workflow                     │
│  complete-cart / capture-payment / fulfill-order   │
└──────────────────────────┬─────────────────────────┘
                           │
           ┌───────────────▼──────────────────┐
           │       ILockingModule             │
           │  acquire / release / execute     │
           └──┬──────────────────────────┬───┘
              │                          │
   ┌──────────▼──────────┐   ┌──────────▼──────────────┐
   │  InProcessProvider   │   │  RedisProvider           │
   │  (Map mutex chain)   │   │  (Redlock algorithm)     │
   └─────────────────────┘   └─────────────────────────┘
                               ┌────────────────────────┐
                               │  PostgreSQLProvider     │
                               │  (advisory locks)       │
                               └────────────────────────┘
```

---

## 4. Solution Strategy

- **`execute()` as primary API**: Hides acquire/release complexity; guarantees release via `finally`.
- **Three providers covering the deployment spectrum**: in-process (dev), PostgreSQL (staging / no Redis), Redis (production).
- **TTL on all locks**: Prevents permanent deadlocks from crashed processes.
- **Lexicographic multi-key ordering**: Prevents deadlocks when multiple keys are acquired simultaneously.

---

## 5. Building Block View

### Level 1

```
LockingModule
  ├── ILockingModule              (interface: acquire, release, execute)
  ├── InProcessLockingProvider   (Map-based, single-process)
  ├── RedisLockingProvider       (Redlock algorithm, multi-process)
  └── PostgreSQLLockingProvider  (advisory locks, multi-process)
```

### Level 2 — execute() contract

```
execute(keys, job, options)
  ├── sort(keys)                  ← lexicographic order (deadlock prevention)
  ├── acquire(keys, { expire })
  ├── try { result = await job() }
  ├── finally { release(keys) }
  └── return result
```

---

## 6. Runtime View

### Scenario: Two Concurrent Inventory Reservations

1. Request A: `execute("inventory:reserve:var_01", reserveA, { expire: 5000 })`.
2. Request B: `execute("inventory:reserve:var_01", reserveB, { expire: 5000 })`.
3. Request A acquires lock (Redis SET NX succeeds).
4. Request B: SET NX fails → enters retry loop (retryCount: 3, delay: 200ms + jitter).
5. Request A completes; releases lock.
6. Request B: retry succeeds, acquires lock, runs job.
7. Both reservations serialised; no double-reservation.

### Scenario: Process Crash During Lock

1. Request A acquires lock (`expire: 5000ms`).
2. Process crashes mid-job.
3. Redis key TTL expires after 5000ms.
4. Request B: lock acquisition succeeds; no deadlock.

---

## 7. Deployment View

| Environment  | Provider    | Infrastructure                    |
|--------------|-------------|-----------------------------------|
| Development  | InProcess   | None                              |
| CI / Testing | InProcess   | None                              |
| Staging      | PostgreSQL  | Existing Medusa DB                |
| Production   | Redis       | Redis 6+ instance or cluster      |

---

## 8. Cross-Cutting Concerns

### Observability
Lock acquisition duration and contention rate should be tracked as application metrics. The Redis provider can emit custom metrics via the `onLockAcquired` / `onLockFailed` hooks.

### Lock Key Management
Lock keys must be scoped to specific resource IDs (not just resource types) to maximise concurrency. Use `{domain}:{operation}:{id}` convention consistently.

---

## 9. Architecture Decisions

| ID  | Decision                                     | Rationale                                                        |
|-----|----------------------------------------------|------------------------------------------------------------------|
| AD1 | `execute()` as primary API surface           | Always-release guarantee without boilerplate at call sites       |
| AD2 | Three provider tiers                         | Covers dev (no infra), staging (PG only), prod (Redis)           |
| AD3 | TTL required on every lock acquisition       | Prevents permanent deadlocks as a hard safety guarantee          |
| AD4 | Lexicographic key ordering in multi-key lock | Standard deadlock prevention; no custom deadlock detection needed |

---

## 10. Quality Scenarios

| Quality      | Scenario                                            | Measure                                      |
|--------------|-----------------------------------------------------|----------------------------------------------|
| Correctness  | 100 concurrent cart completions for 1 item in stock | Exactly 1 succeeds; 99 fail with stock error |
| Safety       | Worker process killed mid-lock                      | Lock auto-releases after TTL (< 5s)          |
| Simplicity   | Developer adds lock to a new critical section       | 2-line code change; no infrastructure setup  |
| Replace      | Config changed to Redis + restart                   | Distributed locking active with no code changes |
