# Workflow Engine Redis Module

## Overview

The `workflow-engine-redis` module is Medusa's production-grade workflow execution engine. It persists all workflow execution state to Redis, supports distributed execution across multiple worker processes, and provides retry and timeout handling for long-running commerce operations.

## Purpose

Production Medusa deployments scale horizontally: multiple API servers handle HTTP requests while dedicated worker processes execute background workflows. The Redis workflow engine coordinates execution across these processes — ensuring a workflow started by an API server can be picked up and continued by any available worker, and that its state survives process restarts.

## Key Features

- **Persistent state** — workflow execution state (step results, compensations, transaction context) is serialised and stored in Redis.
- **Distributed execution** — any worker process can execute pending workflow steps; workers compete for work via Redis lists.
- **Long-running workflows** — workflows can span multiple ticks/processes; async steps (e.g., waiting for a webhook) are fully supported.
- **Retry handling** — steps configured with `maxRetries` are automatically retried on failure with configurable backoff.
- **Timeout handling** — steps with `timeout` set will be marked as failed if they exceed their time limit.
- **Dedicated worker mode** — set `WORKER_MODE=worker` (or `WORKER_MODE=shared`) to run a process that only consumes workflow tasks without serving HTTP.
- **Same interface** — satisfies `IWorkflowEngineService`; transparent swap for `workflow-engine-inmemory`.

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
      resolve: "@medusajs/workflow-engine-redis",
      key: Modules.WORKFLOW_ENGINE,
      options: {
        redis: {
          url: process.env.REDIS_URL,
        },
      },
    },
  ],
})
```

## Worker Mode

```bash
# Start a dedicated background worker
WORKER_MODE=worker node .medusa/server/src/main.js

# Start a combined API + worker (not recommended for scale)
WORKER_MODE=shared node .medusa/server/src/main.js
```

## Execution Model

```
API Process                   Redis                        Worker Process
──────────────────            ──────────────────           ──────────────────────
run(workflow, input) ──────► HSET workflow:txId state ──► poll queue
                              LPUSH workflow:queue txId ──► execute step 1
                                                            HSET step1 = success
                                                            execute step 2 ...
```

## Limitations

| Constraint | Detail |
|---|---|
| **Redis required** | Needs Redis ≥ 5.0. |
| **Operational complexity** | Requires running dedicated worker processes in production. |
| **Redis memory** | Completed workflow state must be explicitly purged or it accumulates. |

## When to Use

| Scenario | Recommendation |
|---|---|
| Production multi-process deployment | ✅ Required |
| Long-running / async workflows | ✅ Required |
| Workflow retry / timeout | ✅ Use this |
| Local development / tests | ❌ Use `workflow-engine-inmemory` |

## Related Modules

- [`workflow-engine-inmemory`](../workflow-engine-inmemory/README.md) — development alternative.
- [`event-bus-redis`](../event-bus-redis/README.md) — typically paired with this in production.
