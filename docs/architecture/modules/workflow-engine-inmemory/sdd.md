# Software Design Document — Workflow Engine In-Memory

**Module:** `@medusajs/workflow-engine-inmemory`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `workflow-engine-inmemory` module implements `IWorkflowEngineService` using an in-process state machine. Workflow execution state is maintained in a `Map` keyed by transaction ID, and step compensation is performed synchronously within the same call stack. It is the default engine for development and testing.

---

## 2. Goals and Non-Goals

### Goals
- Provide a fully functional workflow engine requiring no external services.
- Support the full step lifecycle: idle → invoking → done / failed → compensating → reverted.
- Execute compensations synchronously on step failure.
- Support workflow hooks and subscriber notifications.

### Non-Goals
- Persistence across process restarts.
- Distributed execution across multiple processes.
- Scheduled retries with delays.

---

## 3. Architecture

### 3.1 Component Model

```
┌──────────────────────────────────────────────────────────┐
│                  Medusa Container (DI)                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │          WorkflowEngineInMemoryService             │  │
│  │  ──────────────────────────────────────────────    │  │
│  │  + run(workflowId, options)                        │  │
│  │  + setStepSuccess(options)                         │  │
│  │  + setStepFailure(options)                         │  │
│  │  + getRunningTransaction(wfId, txId)               │  │
│  │  + subscribe / unsubscribe                         │  │
│  │                                                    │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  DistributedTransactionOrchestrator          │  │  │
│  │  │  (shared utility from workflows-sdk)         │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                                                    │  │
│  │  ┌─────────────────────────────────────────────┐   │  │
│  │  │  TransactionStateMap                        │   │  │
│  │  │  Map<txId, DistributedTransactionState>     │   │  │
│  │  └─────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `WorkflowEngineInMemoryService` | Implements `IWorkflowEngineService`; delegates to orchestrator |
| `DistributedTransactionOrchestrator` | (shared) State machine driving step execution and compensation |
| `DistributedTransaction` | Value object representing one workflow execution instance |
| `TransactionStateMap` | In-memory `Map<string, DistributedTransactionState>` |

### 3.3 Transaction Lifecycle

```
run()
  │
  ▼
[not_started]
  │  begin()
  ▼
[invoking]
  │  step 1 success
  ▼
[invoking]
  │  step 2 failure
  ▼
[compensating]     ◄─── compensation invoked for step 1
  │
  ▼
[reverted]

  OR (all steps success)
[invoking] ──► [done]
```

### 3.4 Step Execution Flow

1. `run(workflowId, { input })` creates a `DistributedTransaction` with a new `transactionId`.
2. The orchestrator retrieves the workflow definition (steps array, step options).
3. Steps are executed in order; each step's `invoke` function receives `(input, { container })`.
4. `StepResponse(result, compensationData)` is stored in the transaction state.
5. On failure, the orchestrator walks backwards through completed steps calling each `compensate(compensationData, { container })`.

---

## 4. Data Structures

```typescript
interface DistributedTransactionState {
  transactionId: string
  workflowId: string
  status: TransactionStatus
  steps: Map<string, StepState>
  context: Record<string, unknown>
}

type TransactionStatus =
  | "not_started" | "invoking" | "done"
  | "compensating" | "reverted" | "failed"

interface StepState {
  stepId: string
  status: "idle" | "invoking" | "done" | "compensating" | "reverted" | "failed"
  output?: unknown
  compensationData?: unknown
}
```

---

## 5. Error Handling

- Uncaught exceptions in a step's `invoke` function transition the step to `failed` and trigger the compensation chain.
- Uncaught exceptions in a step's `compensate` function are logged; compensation continues for remaining steps.
- After all compensations complete, the transaction transitions to `reverted`.

---

## 6. Dependencies

| Dependency | Purpose |
|---|---|
| `@medusajs/framework/workflows-sdk` | `DistributedTransactionOrchestrator`, `createWorkflow`, `createStep` |
| `@medusajs/framework/types` | `IWorkflowEngineService` |
| `@medusajs/framework/utils` | Module helpers |

---

## 7. Testing Strategy

- Unit tests for each transaction lifecycle transition.
- Integration tests run real Medusa core-flow workflows (e.g., `createOrderWorkflow`) and verify step outputs and compensation.
- All Medusa HTTP integration tests use `workflow-engine-inmemory` by default.
