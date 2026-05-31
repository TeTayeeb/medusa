# SpecKit — Workflow Engine In-Memory

**Module:** `@medusajs/workflow-engine-inmemory`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/workflow-engine-inmemory` |
| DI key | `Modules.WORKFLOW_ENGINE` |
| Interface | `IWorkflowEngineService` |
| Category | Infrastructure / Workflow Orchestration |
| Default for | Development and testing |

---

## 2. Functional Specifications

### SPEC-WEI-001 — Successful workflow completes
**Given** a workflow with steps S1 → S2 → S3 where all steps succeed
**When** `engine.run(workflowId, { input })` is called
**Then** the returned result contains the output of the last step.
**And** the transaction status is `"done"`.

### SPEC-WEI-002 — Step failure triggers compensation
**Given** a workflow with steps S1 → S2 → S3 where S3 fails
**When** `engine.run(workflowId, { input })` is called
**Then** compensation functions are invoked in reverse order: compensate(S2), compensate(S1).
**And** the transaction status is `"reverted"`.
**And** `run()` rejects with the original error from S3.

### SPEC-WEI-003 — First step failure triggers no compensation
**Given** a workflow with steps S1 → S2 where S1 fails immediately
**When** `engine.run(workflowId, { input })` is called
**Then** no compensation functions are invoked (S1 has no prior successful step to undo).
**And** the transaction status is `"reverted"`.

### SPEC-WEI-004 — Compensation failure does not stop chain
**Given** S2 compensation throws an exception
**When** the compensation chain runs
**Then** S1 compensation is still invoked.
**And** S2's compensation failure is logged.

### SPEC-WEI-005 — Idempotent re-run
**Given** a workflow with `transactionId: "txn-abc"` has completed
**When** `engine.run(workflowId, { transactionId: "txn-abc", input })` is called again
**Then** steps are NOT re-executed.
**And** the original result is returned.

### SPEC-WEI-006 — getRunningTransaction
**When** `engine.getRunningTransaction(workflowId, transactionId)` is called for an active transaction
**Then** a `DistributedTransaction` object is returned reflecting current step states.

### SPEC-WEI-007 — setStepSuccess / setStepFailure
**Given** a workflow with an async step (returns `StepResponse` with `isPermanentFailure: false`)
**When** `engine.setStepSuccess({ transactionId, stepId, response })` is called externally
**Then** the workflow resumes from that step's success state.

### SPEC-WEI-008 — Hook invocation
**Given** a workflow exposes a hook `"orderCreated"` and a subscriber is registered
**When** the workflow reaches the hook boundary
**Then** the subscriber is invoked with the hook data.

---

## 3. Non-Functional Specifications

### SPEC-WEI-NF-001 — Zero external dependencies
The module must not require Redis, a database, or any external service.

### SPEC-WEI-NF-002 — Interface compatibility
The module must satisfy `IWorkflowEngineService` exactly. All tests written against `workflow-engine-redis` must pass unchanged against `workflow-engine-inmemory`.

### SPEC-WEI-NF-003 — Synchronous compensation
Compensations must be invoked and completed before `run()` rejects.

---

## 4. Configuration Specification

```typescript
// No configuration required.
interface WorkflowEngineInMemoryOptions {}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-WEI-001 | Integration | Required |
| SPEC-WEI-002 | Integration | Required |
| SPEC-WEI-003 | Unit | Required |
| SPEC-WEI-004 | Unit | Required |
| SPEC-WEI-005 | Unit | Required |
| SPEC-WEI-006 | Unit | Required |
| SPEC-WEI-007 | Integration | Required |
| SPEC-WEI-008 | Integration | Required |
| SPEC-WEI-NF-001 | Static analysis | Required |
| SPEC-WEI-NF-002 | Contract tests | Required |
| SPEC-WEI-NF-003 | Unit (assert compensation before rejection) | Required |

---

## 6. Out of Scope

- Persistence across process restarts
- Distributed execution across processes
- Retry scheduling with delays
- Distributed locking
- Worker mode (`WORKER_MODE` env var — that is `workflow-engine-redis`)
