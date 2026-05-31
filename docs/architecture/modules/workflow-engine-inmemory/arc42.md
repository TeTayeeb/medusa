# arc42 Architecture Documentation — Workflow Engine In-Memory

**Module:** `@medusajs/workflow-engine-inmemory`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The workflow-engine-inmemory module must execute Medusa's distributed transaction workflows entirely in-process, without external infrastructure. It must drive the full step lifecycle (invoke → done/failed → compensate → reverted), support hooks, and maintain interface compatibility with `workflow-engine-redis`.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Completeness** | All workflow step lifecycle states are reachable |
| 2 | **Simplicity** | Works with zero config in development |
| 3 | **Test isolation** | Each test run starts with a clean, empty state store |

---

## 2. Architecture Constraints

- Must use `DistributedTransactionOrchestrator` from `@medusajs/framework/workflows-sdk`.
- Must not introduce persistence (DB or Redis).
- Must be swappable with `workflow-engine-redis` without changes to workflows or steps.

---

## 3. System Scope and Context

```
┌──────────────────────────────────────────────────────────────────┐
│                      Medusa Application                          │
│                                                                  │
│   createOrderWorkflow.run({ input })                             │
│     │                                                            │
│     ▼                                                            │
│   ┌─────────────────────────────────────────────────────┐       │
│   │   workflow-engine-inmemory                          │       │
│   │   WorkflowEngineInMemoryService                     │       │
│   │   DistributedTransactionOrchestrator                │       │
│   │   TransactionStateMap (in-process memory)           │       │
│   └─────────────────────────────────────────────────────┘       │
│                                                                  │
│   [No external systems]                                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

The orchestrator drives workflow execution as a synchronous state machine. Each `run()` call allocates a new `DistributedTransaction` in the `TransactionStateMap`, executes steps in order, and on failure walks the compensations in reverse. All of this happens within the same async call stack as the original `run()` invocation.

---

## 5. Building Blocks

```
workflow-engine-inmemory
  └── WorkflowEngineInMemoryService  (IWorkflowEngineService impl)
        ├── DistributedTransactionOrchestrator  (from workflows-sdk)
        │     ├── Step executor
        │     └── Compensation chain walker
        └── TransactionStateMap  (Map<txId, DistributedTransactionState>)
```

---

## 6. Runtime View

### Scenario: Successful Workflow

```
run("createOrder", input)
  │
  ▼  allocate txId → TransactionStateMap
  │
  ├─► Step 1: reserveInventory.invoke(input)  → StepResponse(ok, data1)
  ├─► Step 2: createOrderRecord.invoke(input)  → StepResponse(ok, data2)
  └─► Step 3: capturePayment.invoke(input)     → StepResponse(ok, data3)
        │
        ▼
  Transaction status: "done"
  run() resolves with { result, transaction }
```

### Scenario: Compensation

```
run("createOrder", input)
  ├─► Step 1 ✓  (stored compensation data)
  ├─► Step 2 ✓
  └─► Step 3 ✗  THROWS
        │
        ▼  compensation chain (reverse order)
  compensate(Step 2, data2)
  compensate(Step 1, data1)
  Transaction status: "reverted"
  run() rejects with original error
```

---

## 7. Deployment View

No external deployment. Runs embedded in the Medusa process. The `TransactionStateMap` is a process-local heap object, reset on restart.

---

## 8. Cross-Cutting Concepts

### Idempotency
Each `run()` call with a unique `transactionId` creates a fresh transaction. Re-running the same `transactionId` will find the existing completed transaction and return its result without re-executing steps.

### Hooks
Workflows can expose named hooks. Hook handlers registered via `subscribe()` are stored in-memory alongside subscriber registrations and invoked at the appropriate step boundary.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Delegate orchestration to `DistributedTransactionOrchestrator` | Shared logic with redis engine; avoids duplication |
| AD-2 | No background retry scheduler | Retries are immediate; simplicity preferred for dev |
| AD-3 | In-memory state only | Process isolation makes each test run deterministic |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Full lifecycle coverage | Unit tests for each TransactionStatus transition |
| Compensation correctness | Integration tests verify steps undone in reverse order |
| Hook invocation | Integration tests verify hook handlers called at correct step boundary |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| State loss on crash | Low (dev only) | By design; use `workflow-engine-redis` in production |
| Memory growth from uncleaned transactions | Low | Transactions are removed from map after completion |
| No distributed locking | Low (dev only) | By design; single process cannot race with itself |
