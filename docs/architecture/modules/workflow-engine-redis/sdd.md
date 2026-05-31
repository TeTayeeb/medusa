# Software Design Document — Workflow Engine Redis

**Module:** `@medusajs/workflow-engine-redis`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `workflow-engine-redis` module implements `IWorkflowEngineService` with Redis as the state persistence and task queue backend. It is built on top of BullMQ (or a custom Redis-backed queue) to support distributed execution, long-running workflows, retries, and timeout handling.

---

## 2. Goals and Non-Goals

### Goals
- Persist workflow execution state durably in Redis.
- Support distributed execution: any available worker can execute any pending step.
- Provide configurable retry policies and step timeouts.
- Support long-running asynchronous workflows (e.g., waiting for external webhooks).
- Enable dedicated worker processes (`WORKER_MODE=worker`).

### Non-Goals
- Workflow scheduling / cron (separate concern).
- Cross-datacenter replication.
- Infinite workflow history retention.

---

## 3. Architecture

### 3.1 Component Model

```
API Process                     Redis                         Worker Process
──────────────────────          ──────────────────────        ───────────────────────────
WorkflowEngineRedisService      ┌─ workflow:state ─────┐     WorkflowEngineRedisService
  run(workflowId, input)        │  HSET txId state json │       DistributedTransactionOrchestrator
    └─► enqueue step 1   ──────►│  LPUSH queue txId:s1  │──►     dequeue step 1
                                └──────────────────────┘       invoke step 1
                                                                HSET txId:s1 done
                                                                enqueue step 2
                                                              invoke step 2 ...
                                                              HSET txId done
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `WorkflowEngineRedisService` | Implements `IWorkflowEngineService`; coordinates state and queue |
| `RedisDistributedTransaction` | Wraps `DistributedTransaction` with Redis-persisted state |
| `WorkflowStepQueue` | Redis List or BullMQ queue of pending step execution tasks |
| `WorkflowWorker` | Background polling loop; dequeues and executes steps |
| `RetryPolicy` | Applies delay + max-retry logic for failed steps |

### 3.3 State Serialisation

Transaction state is stored as JSON in a Redis Hash:

```
HSET workflow:transactions <txId> <json>
```

Step state is stored per-step:
```
HSET workflow:steps <txId>:<stepId> <json>
```

### 3.4 Worker Mode

`WORKER_MODE` environment variable controls process behaviour:

| Value | Behaviour |
|---|---|
| `server` (default) | Process serves HTTP only; does not consume workflow tasks |
| `worker` | Process consumes workflow tasks only; does not serve HTTP |
| `shared` | Process serves HTTP and consumes workflow tasks |

### 3.5 Retry Policy

Steps declare retry options in their definition:

```typescript
const myStep = createStep(
  { name: "my-step", maxRetries: 3, retryInterval: 2000 },
  async (input, { container }) => { ... }
)
```

On failure, the step is re-enqueued with a delay of `retryInterval * attempt` (linear backoff). After `maxRetries` exhaustion, the compensation chain is triggered.

### 3.6 Timeout Handling

Steps with a `timeout` (milliseconds) are wrapped in a `Promise.race`:

```typescript
await Promise.race([
  step.invoke(input, context),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Step timeout")), step.timeout)
  ),
])
```

On timeout, the step transitions to `failed` and the compensation chain begins.

---

## 4. Data Structures

```typescript
interface WorkflowEngineRedisOptions {
  redis: {
    url: string
    options?: RedisOptions
  }
}

// Stored in Redis per transaction
interface PersistedTransactionState {
  transactionId: string
  workflowId: string
  status: TransactionStatus
  context: Record<string, unknown>
  createdAt: string
  updatedAt: string
}
```

---

## 5. Error Handling

- **Step failure** — transitions step to `failed`; triggers compensation; compensation failures are logged and continue.
- **Redis connection loss** — BullMQ/custom poller retries with exponential backoff; in-flight steps may be re-queued.
- **Stale locks** — a heartbeat mechanism detects workers that died mid-step; orphaned tasks are re-queued after a TTL.

---

## 6. Dependencies

| Dependency | Purpose |
|---|---|
| `ioredis` | Redis connectivity |
| `@medusajs/framework/workflows-sdk` | `DistributedTransactionOrchestrator` |
| `@medusajs/framework/types` | `IWorkflowEngineService` |

---

## 7. Testing Strategy

- Unit tests mock Redis and assert correct HSET/LPUSH/BRPOP calls per lifecycle transition.
- Integration tests use a real Redis instance and verify: end-to-end workflow completion, retry after simulated step failure, compensation chain on unrecoverable failure, worker handoff between two processes.
