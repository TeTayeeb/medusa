# arc42 Architecture Documentation — Workflow Engine Redis

**Module:** `@medusajs/workflow-engine-redis`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The workflow-engine-redis module must persist workflow execution state in Redis, support distributed step execution across multiple worker processes, provide retry and timeout handling, and maintain full interface compatibility with `workflow-engine-inmemory`.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Durability** | Workflow state survives an API process crash mid-execution |
| 2 | **Scalability** | Multiple dedicated worker processes execute steps concurrently |
| 3 | **Correctness** | At-most-once step execution per workflow instance (no duplicate side effects) |

---

## 2. Architecture Constraints

- Must use Redis for state persistence (no separate database table).
- Must use `DistributedTransactionOrchestrator` from `@medusajs/framework/workflows-sdk`.
- `WORKER_MODE` env var must control whether a process consumes workflow tasks.
- Must be swappable with `workflow-engine-inmemory` without changing workflow or step definitions.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────────────────┐
│ External: Redis                                                  │
│   Hashes: workflow:transactions, workflow:steps                  │
│   Lists:  workflow:queue (pending step tasks)                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ ioredis
       ┌───────────────────┴───────────────────┐
       │ API Process (WORKER_MODE=server)       │
       │   run() → persist state → enqueue     │
       └───────────────────────────────────────┘
                           │
       ┌───────────────────▼───────────────────┐
       │ Worker Process (WORKER_MODE=worker)    │
       │   dequeue → execute step → persist    │
       └───────────────────────────────────────┘
```

---

## 4. Solution Strategy

Workflow state is serialised to Redis Hashes at every lifecycle transition. Pending step execution is modelled as a Redis List (queue); workers `BRPOP` tasks and execute the corresponding step. Distributed locking via a Redis key prevents two workers from executing the same step simultaneously. Retry scheduling re-enqueues tasks after a delay using `EXPIRE` on a scheduling key.

---

## 5. Building Blocks

```
workflow-engine-redis
  ├── WorkflowEngineRedisService   (IWorkflowEngineService impl)
  ├── RedisDistributedTransaction  (persisted transaction wrapper)
  ├── WorkflowStepQueue            (Redis List / BRPOP)
  ├── WorkflowWorker               (poll loop)
  ├── DistributedLock              (Redis SET NX EX)
  └── RetryPolicy                  (linear backoff, re-enqueue)
```

---

## 6. Runtime View

### Scenario: Cross-Process Workflow

```
API (run)                         Redis                          Worker
  │                                 │                              │
  ├─ persist transaction state ───► HSET workflow:tx abc pending   │
  ├─ enqueue step 1 ──────────────► LPUSH workflow:queue abc:s1    │
  │                                 │                              │
  │                                 │ ◄── BRPOP workflow:queue ───┤
  │                                 │ ──► abc:s1 ────────────────►│
  │                                 │         HSET abc:s1 invoking │
  │                                 │         execute step 1       │
  │                                 │         HSET abc:s1 done     │
  │                                 │         LPUSH queue abc:s2   │
```

---

## 7. Deployment View

```
┌──────── Production Kubernetes ───────────────────────────────────┐
│                                                                  │
│  ┌──────────────────┐   ┌──────────────────┐  ┌──────────────┐  │
│  │  API Pods ×3      │   │  Worker Pods ×2   │  │  Redis       │  │
│  │  WORKER_MODE=server│  │  WORKER_MODE=worker│ │  (HA Cluster)│  │
│  └──────────────────┘   └──────────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### Distributed Locking
Before executing a step, a worker acquires a lock: `SET lock:abc:s1 workerId NX EX 30`. If the lock cannot be acquired (another worker is executing the step), the task is re-queued. The lock is released after step completion or expiry.

### Retry and Timeout
Retry count and interval are declared per-step in the workflow definition. On failure, the orchestrator increments a retry counter stored in the step's Redis hash entry. If `retryCount < maxRetries`, the step is re-enqueued after `retryInterval × attempt` ms.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Redis Hash for state | Efficient partial updates; no need to read-modify-write entire objects |
| AD-2 | `WORKER_MODE` env var | Allows same Docker image in API and worker roles |
| AD-3 | Distributed lock per step | Prevents duplicate side effects from concurrent workers |
| AD-4 | BRPOP for task queue | Blocking pop avoids busy-waiting; native to Redis |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Durability | Integration test: kill API mid-workflow, worker completes it |
| No duplicates | Integration test: two workers, one step — only one executes |
| Retry correctness | Integration test: step fails N times, succeeds on retry N+1 |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Redis unavailability | High | Use Redis Cluster + Sentinel |
| Lock expiry during slow step | Medium | Tune EX timeout higher than max step duration |
| Redis memory growth | Medium | Purge completed transaction state after TTL |
| Worker starvation | Low | Scale worker pods; monitor queue depth |
