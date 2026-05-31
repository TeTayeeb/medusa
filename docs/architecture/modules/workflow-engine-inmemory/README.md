# Workflow Engine In-Memory Module

## Overview

The `workflow-engine-inmemory` module is Medusa's in-process workflow execution engine. It stores all workflow execution state in memory and runs compensation (rollback) logic synchronously. It is the default engine for development and testing, requiring no external infrastructure.

## Purpose

Medusa's commerce operations — order creation, payment capture, fulfilment — are modelled as durable workflows composed of discrete, compensable steps. The workflow engine is responsible for orchestrating those steps, tracking their state, and invoking compensations if a step fails. The in-memory implementation fulfils this role within a single process, making it easy to run and test locally.

## Key Features

- **In-process execution** — steps run directly in the Node.js event loop; no separate worker process needed.
- **Synchronous compensation** — on step failure, all previously executed step compensations are invoked in reverse order within the same request context.
- **Idempotent step execution** — each step is assigned a unique transaction ID; re-running a workflow with the same ID will skip already-completed steps.
- **Hooks support** — workflows can expose named hooks that subscribers can extend at runtime.
- **Zero dependencies** — no Redis, no database table required; state is held in a `Map<transactionId, WorkflowTransactionState>`.

## Interface

```typescript
interface IWorkflowEngineService {
  run(workflowId: string, options: WorkflowRunOptions): Promise<WorkflowResult>
  getRunningTransaction(workflowId: string, transactionId: string, options?: TransactionOptions): Promise<DistributedTransaction>
  setStepSuccess(options: SetStepSuccessOptions): Promise<WorkflowResult>
  setStepFailure(options: SetStepFailureOptions): Promise<WorkflowResult>
  subscribe(options: WorkflowSubscribeOptions): void
  unsubscribe(options: WorkflowUnsubscribeOptions): void
}
```

## Configuration

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/workflow-engine-inmemory",
      key: Modules.WORKFLOW_ENGINE,
    },
  ],
})
```

This is the default when `NODE_ENV=development` and no workflow engine is explicitly configured.

## Execution Model

```
run(workflowId, input)
  │
  ├─ Step 1 ──► success ──► Step 2 ──► success ──► Step 3 ──► success ──► DONE
  │
  └─ Step 1 ──► success ──► Step 2 ──► FAILURE
                                │
                                └─ compensate(Step 1) ──► ROLLED BACK
```

## Limitations

| Constraint | Detail |
|---|---|
| **No persistence** | Workflow state is lost on process restart. Long-running workflows cannot survive a restart. |
| **Single process only** | A workflow started on one process cannot be observed or continued by another. |
| **No retry scheduler** | Retries are attempted immediately, not scheduled for a future time. |
| **No distributed locking** | Two processes could attempt the same workflow concurrently without coordination. |

## When to Use

| Scenario | Recommendation |
|---|---|
| Local development | ✅ Default choice |
| Unit / integration tests | ✅ Ideal |
| Short-lived, synchronous workflows | ✅ Acceptable |
| Long-running or asynchronous workflows | ❌ Use `workflow-engine-redis` |
| Multi-process deployments | ❌ Use `workflow-engine-redis` |

## Related Modules

- [`workflow-engine-redis`](../workflow-engine-redis/README.md) — production-grade alternative.
- [`event-bus-local`](../event-bus-local/README.md) — typically paired with this module in development.
