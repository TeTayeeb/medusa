# `@medusajs/framework/workflows-sdk` — Workflow Composition SDK

**Version:** 2.15.4  
**License:** MIT  
**Category:** Core Infrastructure — Workflow Engine

---

## Overview

`@medusajs/framework/workflows-sdk` (distributed as part of `@medusajs/framework`, source in `packages/core/workflows-sdk`) is the low-level SDK for composing **distributed, compensable workflows** in Medusa v2.

It exposes a small, composable API — `createStep`, `createWorkflow`, `transform`, `when`, `parallelize`, `createHook` — that lets you build complex multi-step business processes with automatic rollback, conditional branching, and parallel execution, all described in pure TypeScript.

---

## Core Concepts

### Step

A **step** is the atomic unit of a workflow. It has two functions:

1. **Invoke** — performs the side-effect (write to DB, call an API, send an email).
2. **Compensate** *(optional)* — undoes the invoke on workflow failure (delete DB row, cancel API call).

Steps are composable: their outputs become the typed inputs to subsequent steps.

### Workflow

A **workflow** is a directed acyclic graph (DAG) of steps declared in a composer function. The composer function runs at **definition time** — it builds the execution graph but does not execute steps immediately. Actual execution happens when `.run()` is called on the instantiated workflow.

### WorkflowData

`WorkflowData<T>` is a lazy reference to a value that will be resolved at execution time. You cannot read `.value` from it inside the composer — use `transform()` to manipulate it.

---

## API Reference

### `createStep`

```typescript
function createStep<TInput, TOutput, TCompensateInput>(
  nameOrConfig: string | StepConfig,
  invokeFn: (input: TInput, ctx: StepExecutionContext) => StepResponse<TOutput, TCompensateInput>,
  compensateFn?: (input: TCompensateInput | undefined, ctx: StepExecutionContext) => unknown
): StepFunction<TInput, TOutput>
```

**Example:**

```typescript
import { createStep, StepResponse } from "@medusajs/framework/workflows-sdk"
import { Modules } from "@medusajs/framework/utils"

export const reserveStockStep = createStep(
  "reserve-stock-step",
  async (input: { variant_id: string; quantity: number }, { container }) => {
    const inventoryModule = container.resolve(Modules.INVENTORY)
    const [reservation] = await inventoryModule.createReservationItems([input])
    return new StepResponse(reservation, reservation.id)
  },
  async (reservationId: string | undefined, { container }) => {
    if (!reservationId) return
    const inventoryModule = container.resolve(Modules.INVENTORY)
    await inventoryModule.deleteReservationItems([reservationId])
  }
)
```

### `createWorkflow`

```typescript
function createWorkflow<TInput, TOutput, THooks>(
  nameOrConfig: string | WorkflowConfig,
  composerFn: (input: WorkflowData<TInput>) => WorkflowResponse<TOutput>
): ReturnWorkflow<TInput, TOutput, THooks>
```

**Example:**

```typescript
import { createWorkflow, WorkflowResponse } from "@medusajs/framework/workflows-sdk"
import { reserveStockStep } from "./steps/reserve-stock"
import { notifyCustomerStep } from "./steps/notify-customer"

export const reserveAndNotifyWorkflow = createWorkflow(
  "reserve-and-notify",
  (input: { variant_id: string; quantity: number; customer_email: string }) => {
    const reservation = reserveStockStep(input)
    notifyCustomerStep({ email: input.customer_email, reservation_id: reservation.id })
    return new WorkflowResponse(reservation)
  }
)

// Usage
const { result } = await reserveAndNotifyWorkflow(req.scope).run({
  input: { variant_id: "var_01", quantity: 2, customer_email: "user@example.com" },
})
```

### `transform`

Transforms step outputs inside the composer without accessing runtime values directly:

```typescript
import { transform } from "@medusajs/framework/workflows-sdk"

const orderSummary = transform(
  { order, customer },
  ({ order, customer }) => ({
    id: order.id,
    customer_name: `${customer.first_name} ${customer.last_name}`,
    total: order.total,
  })
)
```

### `when`

Conditional step execution — replaces `if` statements inside the composer:

```typescript
import { when } from "@medusajs/framework/workflows-sdk"

const taxLines = when(input, (i) => i.requires_tax_calculation).then(() =>
  calculateTaxLinesStep(input)
)
```

### `parallelize`

Runs multiple steps concurrently and returns their results as a typed tuple:

```typescript
import { parallelize } from "@medusajs/framework/workflows-sdk"

const [variant, price, media] = parallelize(
  createVariantStep(variantInput),
  createPriceStep(priceInput),
  uploadMediaStep(mediaInput)
)
```

### `createHook`

Defines an extension point that external code can attach handlers to:

```typescript
import { createHook, WorkflowResponse } from "@medusajs/framework/workflows-sdk"

const productCreated = createHook("productCreated", { product })
return new WorkflowResponse(product, { hooks: [productCreated] })

// External registration
myWorkflow.hooks.productCreated(async ({ product }, { container }) => {
  // send notification, update search index, etc.
})
```

### `StepResponse`

```typescript
// result only (compensation receives the same value)
return new StepResponse(result)

// result + separate compensation data
return new StepResponse(result, compensationData)
```

### `WorkflowResponse`

```typescript
// simple output
return new WorkflowResponse(output)

// output + hooks
return new WorkflowResponse(output, { hooks: [hook1, hook2] })
```

---

## Execution Model

```
workflow(scope).run({ input })
  │
  ├─ Step 1 invoke   → StepResponse(result1, compensate1)
  ├─ Step 2 invoke   → StepResponse(result2, compensate2)
  ├─ Step 3 invoke   ✗ throws
  │     │
  │     └─ Step 3 compensate(compensate3)
  │     └─ Step 2 compensate(compensate2)
  │     └─ Step 1 compensate(compensate1)
  │
  └─ Error propagated to caller
```

The execution engine (`@medusajs/orchestration`) manages the transaction lifecycle. It can run in-memory (default, synchronous) or backed by a Redis/database store for durable async workflows.

---

## Type Helpers

| Type | Description |
|---|---|
| `WorkflowData<T>` | Lazy workflow value reference |
| `StepExecutionContext` | `{ container, context, metadata }` available in every step |
| `InvokeFn<TInput, TOutput, TCompensateInput>` | Invoke function type |
| `CompensateFn<T>` | Compensation function type |
| `ReturnWorkflow<TInput, TOutput, THooks>` | Callable workflow with `.run()` and `.hooks` |

---

## Related Packages

- [`@medusajs/core-flows`](../core-flows/README.md) — built-in workflows using this SDK
- [`@medusajs/orchestration`](https://github.com/medusajs/medusa) — underlying transaction engine
- [`@medusajs/framework`](../framework/README.md) — runtime that executes workflows
