# arc42 Architecture Document — `@medusajs/framework/workflows-sdk`

**Version:** 2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The Workflows SDK must:

- Enable type-safe composition of multi-step business processes in TypeScript.
- Guarantee automatic compensation (rollback) when any step fails.
- Support conditional execution and parallel step scheduling declaratively.
- Allow step reuse across multiple workflows.
- Be testable in isolation without a running database or HTTP server.
- Integrate with a pluggable execution engine (in-memory or Redis-backed).

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Correctness** | If step 5 of 8 fails, steps 4 through 1 compensate in order, leaving no partial state. |
| 2 | **Type Safety** | Passing a wrong type between steps fails at compile time, not runtime. |
| 3 | **Composability** | A step written for one workflow is reused in five others without modification. |
| 4 | **Testability** | A step is unit-tested with a mocked container in under 50ms. |
| 5 | **Engine Agnosticism** | The same workflow runs in-memory in tests and durably in production. |

### 1.3 Stakeholders

| Role | Expectation |
|---|---|
| Workflow Author | Minimal boilerplate; excellent TypeScript inference |
| Plugin Developer | Stable step/workflow contracts; reliable hooks |
| Platform Engineer | Durable execution; observable transaction state |
| QA / Test Engineer | Unit-testable steps without integration setup |

---

## 2. Architecture Constraints

- The composer function MUST be synchronous and side-effect-free.
- Steps MUST communicate only through `StepResponse` / `WorkflowData`; no shared mutable state.
- Compensation functions MUST be idempotent.
- Workflow IDs MUST be globally unique strings.
- The SDK MUST NOT import from `@medusajs/core-flows` (unidirectional dependency).

---

## 3. System Context

```
┌───────────────────────────────────────────────────────────────┐
│                   @medusajs/framework/workflows-sdk           │
│                                                               │
│   createStep()  ──────────────────────────────────────────┐  │
│   createWorkflow()  ─────────── DAG Builder               │  │
│   transform()   ──────────────── (composition time)       │  │
│   when()        ──────────────────────────────────────────┘  │
│   parallelize() ───────────────── Execution Request          │
│                                         │                     │
│                                         ▼                     │
│                           @medusajs/orchestration             │
│                        (DistributedTransaction engine)        │
│                                         │                     │
│                                         ▼                     │
│                    Step invoke/compensate functions           │
│                    (access MedusaContainer at runtime)        │
└───────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Problem | Solution |
|---|---|
| No `await` / `if` in composer | `WorkflowData<T>` proxy + `when()` + `transform()` |
| Type-safe step chaining | Generic `StepFunction<TInput, TOutput>` |
| Automatic rollback | Compensation stack in orchestration engine |
| Engine swappability | Abstract `IDistributedTransactionStorage` interface |
| Cross-workflow reuse | Steps are standalone functions; import and call anywhere |

---

## 5. Building Blocks (Level 1)

```
workflows-sdk
├── createStep()           — defines an atomic step
├── createWorkflow()       — defines a workflow (DAG builder)
├── StepResponse           — step return value wrapper
├── WorkflowResponse       — workflow return value wrapper
├── transform()            — data manipulation in composer
├── when()                 — conditional execution
├── parallelize()          — concurrent step scheduling
├── createHook()           — extension point declaration
├── useQueryGraphStep      — cross-module read step
└── types/
    ├── WorkflowData<T>    — lazy value reference
    ├── StepExecutionContext — runtime context per step
    ├── InvokeFn           — invoke function type
    ├── CompensateFn       — compensation function type
    └── ReturnWorkflow     — instantiated workflow type
```

---

## 6. Building Blocks (Level 2) — DAG Construction

```
createWorkflow("my-workflow", (input: WorkflowData<Input>) => {
  │
  ├─ step1(input)                    → WorkflowData<Out1>    [node: step1]
  │
  ├─ transform({ out1 }, fn)         → WorkflowData<Out2>    [node: transform-1]
  │
  ├─ when(out2, cond).then(() => {
  │     step2(out2)                  → WorkflowData<Out3?>   [node: step2, guarded]
  │  })
  │
  ├─ parallelize(
  │     step3(out2),                 → WorkflowData<Out4>    [node: step3, parallel]
  │     step4(out2)                  → WorkflowData<Out5>    [node: step4, parallel]
  │  )
  │
  └─ return new WorkflowResponse(out4, { hooks: [hookA] })
})

Resulting DAG edges:
  input → step1 → transform-1 → step2 (conditional)
                              → step3 (parallel)  ┐
                              → step4 (parallel)  ┘ → hookA
```

---

## 7. Runtime View

### 7.1 Workflow Execution Flow

```
workflow(scope).run({ input })

Orchestration Engine:
  1. Begin DistributedTransaction(id, workflowId)
  2. Resolve WorkflowData<Input> → real input value
  3. For each node in topological order:
     a. Resolve input (WorkflowData proxy → real value)
     b. Call step.invokeFn(resolvedInput, ctx)
     c. Store StepResponse in transaction state
     d. Push (stepId, compensationData) onto compensation stack
  4. If no errors: resolve output, fire hooks, return result
  5. If error at step N:
     a. Pop (stepN-1, data), call compensateFn(data, ctx)
     b. Pop (stepN-2, data), call compensateFn(data, ctx)
     …
     c. Rethrow original error
```

### 7.2 Parallel Execution

```
Engine detects parallelize([step3, step4]):
  → Forks two execution contexts
  → Invokes step3 and step4 concurrently (Promise.all)
  → On both complete: merge results into tuple [Out4, Out5]
  → On either fail: cancel sibling, run compensation stack
```

### 7.3 Durable Execution (Redis mode)

```
workflow.run({ input, idempotencyKey: "order-xyz-checkout" })
  → Check Redis: transaction "order-xyz-checkout" exists? Resume from last step.
  → No? Start fresh, checkpoint state after each step.
  → Process crash mid-flight: restart, reload state, resume from last checkpoint.
```

---

## 8. Deployment View

The workflows-sdk is a library with no server component. It runs in the same process as the Medusa application. The execution engine is selected by the Medusa framework at startup based on `projectConfig.redisUrl`:

- **No Redis configured** → `InMemoryDistributedTransactionStorage` (synchronous, non-durable)
- **Redis configured** → `RedisDistributedTransactionStorage` (async, durable, resumable)

---

## 9. Cross-Cutting Concerns

### 9.1 Idempotency

The `idempotencyKey` passed to `.run()` is stored in the transaction. Re-running with the same key returns the cached result without re-executing steps. This protects against double-execution on network retries.

### 9.2 Observability

The `StepExecutionContext.metadata` field is populated by the framework with trace context (OpenTelemetry span IDs) so that step invocations appear as child spans in distributed traces.

---

## 10. Architecture Decisions

### ADR-001: Two-Phase Composition Model

**Context:** Steps must be type-safe and composable without coupling to execution order.  
**Decision:** Separate composition (definition time) from execution (runtime) via the `WorkflowData<T>` proxy pattern.  
**Consequences:** Enables full TypeScript type inference across the DAG; prevents accidental synchronous side-effects in the composer. Trade-off: learning curve for new authors unfamiliar with the deferred execution model.

### ADR-002: Compensation via `StepResponse` Second Argument

**Context:** Compensation functions often need different data than the invoke return value.  
**Decision:** `StepResponse(result, compensationData)` separates the two concerns.  
**Consequences:** Steps can return minimal compensation data (e.g., just an ID) rather than a full DTO, reducing storage overhead in durable mode.

### ADR-003: Hook Execution Outside Transaction

**Context:** Should hooks participate in the compensation chain?  
**Decision:** Hooks execute post-transaction; hook failures do not trigger compensation.  
**Consequences:** Hooks are suitable for non-critical side effects (notifications, search index). For transactional side-effects, use a regular step with a compensation function.
