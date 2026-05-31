# Software Design Document — `@medusajs/framework/workflows-sdk`

**Version:** 2.15.4  
**Status:** Released  
**Authors:** Medusa Core Team

---

## 1. Purpose & Scope

This document describes the internal design of the Medusa Workflow SDK — the primitives for composing distributed, compensable workflows. It covers the execution model, type system, orchestration integration, and the rationale behind key design decisions.

---

## 2. Goals & Non-Goals

### Goals
- Provide an ergonomic TypeScript DSL for workflow composition.
- Guarantee automatic compensation (rollback) on step failure.
- Support conditional (`when`) and parallel (`parallelize`) execution in the same model.
- Be engine-agnostic: work with an in-memory engine for tests and a durable engine in production.
- Enable static type inference of step input/output chains at compile time.

### Non-Goals
- Implement a general-purpose job queue (that is `@medusajs/orchestration`'s concern).
- Provide a UI for workflow visualisation.
- Support non-TypeScript runtimes.

---

## 3. Execution Model

### 3.1 Two-Phase Composition

Workflows are defined in two distinct phases:

**Phase 1 — Composition (definition time):**  
The composer function passed to `createWorkflow()` runs synchronously when the workflow module is loaded. It does not execute steps — it builds a `TransactionStepsDefinition` DAG that describes the order, dependencies, and configuration of each step.

**Phase 2 — Execution (runtime):**  
When `.run({ input })` is called, the orchestration engine traverses the DAG, invokes each step in topological order, passes outputs as inputs, and manages the compensation stack.

This two-phase model is why you cannot use `if` statements or `await` directly in the composer — only the SDK's own `when()`, `transform()`, and `parallelize()` primitives are valid because they operate on `WorkflowData<T>` (lazy references), not resolved values.

### 3.2 WorkflowData Proxy Pattern

`WorkflowData<T>` is implemented as a JavaScript `Proxy`. Property access on it during composition records a "path expression" into the DAG rather than resolving a value. At execution time, the engine replaces the proxy with the real resolved value.

```typescript
// Composition phase — `order` is WorkflowData<OrderDTO>
const order = createOrderStep(input)  // returns WorkflowData<OrderDTO>
const orderId = order.id              // records path: "order.id", not a real string

// Execution phase — engine resolves `order.id` to "ord_01J..."
notifyStep({ order_id: orderId })     // notifyStep receives "ord_01J..."
```

### 3.3 Compensation Stack

Each step's invoke function returns a `StepResponse(result, compensationData)`. The engine pushes `(stepId, compensationData)` onto a stack after each successful invocation. On failure:

1. The engine pops entries from the stack in LIFO order.
2. For each entry, the corresponding compensation function is called with the stored `compensationData`.
3. Compensation continues even if individual compensations fail (all are attempted).
4. The original error is surfaced to the caller after compensation completes.

---

## 4. Component Design

### 4.1 `createStep`

Internally, `createStep` calls `applyStep()` which registers the step's invoke/compensate functions in the current workflow context (a global context stack maintained during composition). The returned `StepFunction<TInput, TOutput>` is a callable that, when invoked in the composer, returns a `WorkflowData<TOutput>` proxy.

```
createStep(id, invokeFn, compensateFn)
  └─ Returns StepFunction<TInput, TOutput>
        └─ When called in composer: registers step in DAG, returns WorkflowData<TOutput>
        └─ When called at runtime: invokeFn(resolvedInput, ctx) → StepResponse
```

### 4.2 `createWorkflow`

`createWorkflow` initialises a new `WorkflowContext` (using a global stack to support nested workflows), runs the composer function to build the DAG, then returns a `ReturnWorkflow` callable.

```typescript
// Simplified internal flow
function createWorkflow(id, composerFn) {
  return function (container) {
    const workflow = new MedusaWorkflow(id)
    pushWorkflowContext(workflow)       // allow steps to register into this workflow
    const result = composerFn(input)    // build DAG (no I/O)
    popWorkflowContext()
    return {
      run: (options) => workflow.run(container, options),
      hooks: workflow.hooks,
      runAsStep: (options) => applyStep({ invokeFn: () => workflow.run(...) })
    }
  }
}
```

### 4.3 `transform`

`transform(values, fn)` registers a pure-function transformation node in the DAG. The `fn` receives resolved values at execution time, not `WorkflowData` proxies. The return value is a new `WorkflowData<TOutput>` that downstream steps can consume.

### 4.4 `when`

`when(values, conditionFn).then(stepsFn)` registers a conditional branch:

1. At composition time, it creates a "condition step" that evaluates `conditionFn(resolvedValues)` at runtime.
2. If the condition returns `true`, the steps returned by `stepsFn()` are executed.
3. If `false`, the steps are skipped and the returned `WorkflowData` resolves to `undefined`.
4. Compensation for conditional steps only runs if the steps were actually executed.

### 4.5 `parallelize`

`parallelize(...steps)` marks the provided steps as having no interdependency. The orchestration engine schedules them concurrently. Their results are collected into a typed tuple, preserving order.

### 4.6 `createHook`

`createHook(name, data)` registers a hook emission point in the DAG. Hooks are treated as special steps that fire **after** the workflow's last regular step. Hook handlers are stored on the workflow object and resolved at runtime via the container.

### 4.7 `StepExecutionContext`

Every invoke and compensate function receives a `StepExecutionContext`:

```typescript
interface StepExecutionContext {
  container: MedusaContainer   // the request-scoped DI container
  context: Context             // shared context (idempotency key, transaction ID)
  metadata: Record<string, unknown>  // step-level metadata
}
```

---

## 5. Type System Design

The SDK makes heavy use of TypeScript conditional types and generics to ensure end-to-end type safety:

- `WorkflowData<T>` propagates the generic type `T` through the DAG.
- `StepFunction<TInput, TOutput>` ensures the step's invoke output type matches the input type expected by downstream steps.
- `ReturnWorkflow<TInput, TOutput, THooks>` types `.run()` input/output and `.hooks.*` registration functions.
- `transform` overloads support up to 6 chained transform functions while preserving type inference.

This means a type error at any point in a workflow (e.g., passing a `string` where a `number` is expected) surfaces as a TypeScript compile error, not a runtime error.

---

## 6. Orchestration Engine Integration

The SDK delegates execution to `@medusajs/orchestration`:

```
workflows-sdk      →    @medusajs/orchestration
createWorkflow()        DistributedTransactionType registration
.run()                  OrchestratorService.beginTransaction()
StepResponse            TransactionStep result storage
compensation stack       TransactionStep.compensate()
```

The SDK supports two execution modes:

| Mode | Engine | State storage | Use case |
|---|---|---|---|
| In-memory (default) | `InMemoryDistributedTransactionStorage` | RAM | Unit tests, synchronous flows |
| Durable (Redis/DB) | `RedisDistributedTransactionStorage` | Redis | Long-running async workflows, retries |

---

## 7. Error Handling

| Scenario | Behaviour |
|---|---|
| Step invocation throws | Compensation stack executes in LIFO order; error rethrown |
| Compensation throws | Logged; remaining compensations continue; no secondary throw |
| `transform` throws | Treated as step failure; compensation runs |
| `when` condition throws | Treated as step failure |
| Hook handler throws | Logged only; does not trigger compensation |

---

## 8. Testing Support

The SDK exposes `__invoke` and `__compensate` methods on step functions (in test environments) to enable direct unit testing without running the full orchestration engine:

```typescript
const result = await reserveStockStep.__invoke(input, mockCtx)
await reserveStockStep.__compensate(result.compensationData, mockCtx)
```
