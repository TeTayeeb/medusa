# SpecKit — Workflow Engine Redis

**Module:** `@medusajs/workflow-engine-redis`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/workflow-engine-redis` |
| DI key | `Modules.WORKFLOW_ENGINE` |
| Interface | `IWorkflowEngineService` |
| Category | Infrastructure / Workflow Orchestration |
| Default for | Production deployments |
| External dependency | Redis ≥ 5.0 |

---

## 2. Functional Specifications

### SPEC-WER-001 — State persisted to Redis on run
**Given** `engine.run("createOrder", { input })` is called
**Then** a Redis Hash entry for the transaction is created: `HGET workflow:transactions <txId>` returns the serialised state.

### SPEC-WER-002 — Workflow survives API process restart
**Given** a workflow is in-flight when the API process is killed
**When** a worker process (or restarted API) polls the queue
**Then** the pending steps are executed and the workflow completes.
**And** no steps that already completed are re-executed (idempotency).

### SPEC-WER-003 — Worker mode controlled by WORKER_MODE
**Given** `WORKER_MODE=worker`
**Then** the process consumes workflow tasks from Redis but does NOT serve HTTP.
**Given** `WORKER_MODE=server`
**Then** the process serves HTTP but does NOT consume workflow tasks.
**Given** `WORKER_MODE=shared`
**Then** the process both serves HTTP and consumes workflow tasks.

### SPEC-WER-004 — Step retry on failure
**Given** a step declares `maxRetries: 2` and fails twice
**When** the step is retried
**Then** it is executed a total of 3 times (initial + 2 retries).
**And** on the third failure, the compensation chain is triggered.

### SPEC-WER-005 — Step timeout
**Given** a step declares `timeout: 5000` (5 seconds) and does not complete within 5 seconds
**Then** the step is marked as `failed` with a timeout error.
**And** the compensation chain is triggered.

### SPEC-WER-006 — Distributed execution: no duplicate steps
**Given** two worker processes are running and one workflow has a pending step
**When** both workers poll simultaneously
**Then** exactly one worker executes the step.
**And** the other worker's execution is a no-op (lock not acquired).

### SPEC-WER-007 — Compensation on unrecoverable failure
**Given** a workflow step fails and exceeds its `maxRetries`
**Then** the compensation chain executes in reverse order.
**And** the final transaction status is `"reverted"`.

### SPEC-WER-008 — getRunningTransaction returns live state
**When** `engine.getRunningTransaction(workflowId, transactionId)` is called
**Then** the returned `DistributedTransaction` reflects the state stored in Redis.

### SPEC-WER-009 — Interface compatibility
The module must satisfy `IWorkflowEngineService` exactly. All tests written against `workflow-engine-inmemory` must pass unchanged against `workflow-engine-redis`.

---

## 3. Non-Functional Specifications

### SPEC-WER-NF-001 — Configurable Redis connection
`options.redis.url` and `options.redis.options` (ioredis passthrough) must be configurable.

### SPEC-WER-NF-002 — Automatic reconnection
On Redis disconnect, the worker must resume polling after reconnection without process restart.

### SPEC-WER-NF-003 — State TTL (cleanup)
Completed transaction state in Redis must be purgeable (e.g., via a configurable retention TTL) to prevent unbounded growth.

---

## 4. Configuration Specification

```typescript
interface WorkflowEngineRedisOptions {
  redis: {
    url: string               // required
    options?: RedisOptions    // ioredis passthrough
  }
}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-WER-001 | Integration (real Redis) | Required |
| SPEC-WER-002 | Integration (kill-and-restart) | Required |
| SPEC-WER-003 | Integration | Required |
| SPEC-WER-004 | Integration | Required |
| SPEC-WER-005 | Integration | Required |
| SPEC-WER-006 | Integration (2-worker race) | Required |
| SPEC-WER-007 | Integration | Required |
| SPEC-WER-008 | Integration | Required |
| SPEC-WER-009 | Contract tests | Required |
| SPEC-WER-NF-001 | Unit | Required |
| SPEC-WER-NF-002 | Integration | Required |
| SPEC-WER-NF-003 | Integration | Required |

---

## 6. Out of Scope

- Workflow scheduling / cron triggers
- Distributed saga pattern (multi-service)
- Cross-datacenter replication
- Workflow history UI / audit log
- Custom step executors / plugins
