# SpecKit — `@medusajs/framework/workflows-sdk`

**Version:** 2.15.4  
**Spec Type:** Behavioural + Interface Specification

---

## 1. Package Identity

| Field | Value |
|---|---|
| Package name | `@medusajs/framework` (sub-path export) |
| Import path | `@medusajs/framework/workflows-sdk` |
| Source package | `@medusajs/workflows-sdk` (`packages/core/workflows-sdk`) |
| Main entry | `dist/index.js` |
| Types entry | `dist/index.d.ts` |

---

## 2. Public API Surface

### 2.1 `createStep`

```typescript
function createStep<TInput, TOutput, TCompensateInput = TOutput>(
  nameOrConfig: string | {
    name: string
    timeout?: number
    maxRetries?: number
    retryInterval?: number
    async?: boolean
    noCompensation?: boolean
  },
  invokeFn: InvokeFn<TInput, TOutput, TCompensateInput>,
  compensateFn?: CompensateFn<TCompensateInput>
): StepFunction<TInput, TOutput>
```

**Step Config options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | required | Unique step ID within workflow |
| `timeout` | `number` | `undefined` | Max ms before step is aborted |
| `maxRetries` | `number` | `0` | Retry count on invocation failure |
| `retryInterval` | `number` | `1000` | Base ms between retries |
| `async` | `boolean` | `false` | Whether step runs asynchronously |
| `noCompensation` | `boolean` | `false` | Skip compensation even if provided |

### 2.2 `createWorkflow`

```typescript
function createWorkflow<TInput, TOutput, THooks extends any[]>(
  nameOrConfig: string | {
    name: string
    store?: boolean    // persist state (durable mode)
    retentionTime?: number  // ms to keep completed transaction
  },
  composerFn: (input: WorkflowData<TInput>) => WorkflowResponse<TOutput>
): ReturnWorkflow<TInput, TOutput, THooks>
```

### 2.3 `StepResponse`

```typescript
class StepResponse<TOutput, TCompensateInput = TOutput> {
  constructor(
    output: TOutput,
    compensateInput?: TCompensateInput
  )
}
```

### 2.4 `WorkflowResponse`

```typescript
class WorkflowResponse<TOutput, THooks extends any[] = any[]> {
  constructor(
    output: WorkflowData<TOutput> | TOutput,
    options?: { hooks?: THooks }
  )
}
```

### 2.5 `transform`

```typescript
function transform<T extends object | WorkflowData, R>(
  values: T,
  transformFn: (resolved: ResolvedType<T>, ctx: StepExecutionContext) => R
): WorkflowData<R>
```

### 2.6 `when`

```typescript
function when<T extends object | WorkflowData>(
  values: T,
  condition: (resolved: ResolvedType<T>, ctx: StepExecutionContext) => boolean
): { then: (resolver: () => WorkflowData<any> | void) => WorkflowData<any> | undefined }
```

### 2.7 `parallelize`

```typescript
function parallelize<T extends (WorkflowData | undefined)[]>(
  ...steps: T
): T
```

### 2.8 `createHook`

```typescript
function createHook<T>(name: string, data: T): WorkflowHook<T>
```

---

## 3. Type Definitions

```typescript
type WorkflowData<T> = {
  // Proxy — cannot be read directly in composer
  // Used only as input to other steps / transform
}

interface StepExecutionContext {
  container: MedusaContainer
  context: {
    transactionId: string
    idempotencyKey?: string
    [key: string]: unknown
  }
  metadata: Record<string, unknown>
}

type InvokeFn<TInput, TOutput, TCompensateInput> = (
  input: TInput,
  ctx: StepExecutionContext
) => StepResponse<TOutput, TCompensateInput> | Promise<StepResponse<TOutput, TCompensateInput>> | void | Promise<void>

type CompensateFn<T> = (
  input: T | undefined,
  ctx: StepExecutionContext
) => unknown | Promise<unknown>

type ReturnWorkflow<TInput, TOutput, THooks> = {
  (container: MedusaContainer): {
    run(options: {
      input?: TInput
      idempotencyKey?: string
      resultFrom?: string
      throwOnError?: boolean
      context?: Record<string, unknown>
    }): Promise<{ result: TOutput; transaction: DistributedTransaction; errors: TransactionStepError[] }>
  }
  hooks: THookRegistry<THooks>
  runAsStep(options: { input: WorkflowData<TInput> }): WorkflowData<TOutput>
}
```

---

## 4. Invariants

1. **Composer purity:** The composer function MUST NOT perform I/O, call `await`, or read `.value` from `WorkflowData`. Violations cause undefined runtime behaviour.
2. **Step ID uniqueness per workflow:** Two steps with the same name in one workflow MUST NOT be registered. The second registration overwrites the first.
3. **Compensation idempotency:** A compensation function MUST NOT throw if called with `undefined` (the invoke may not have run).
4. **WorkflowData opacity:** `WorkflowData<T>` MUST only be passed as input to other SDK functions. Destructuring or spreading it in the composer is invalid.
5. **Hook safety:** A hook handler MUST NOT call `.run()` on the same workflow it's attached to (infinite recursion).
6. **Idempotency key reuse:** Calling `.run()` with an already-completed `idempotencyKey` MUST return the cached result. Steps MUST NOT re-execute.

---

## 5. Error Behaviour

| Scenario | SDK Behaviour |
|---|---|
| Step returns `void` (no `StepResponse`) | Result treated as `undefined`; no compensation data stored |
| Step `invokeFn` returns a rejected Promise | Compensation stack executes; error rethrown |
| `compensateFn` is `undefined` | Step is registered as `noCompensation: true`; skipped during rollback |
| `maxRetries > 0` and all retries fail | Compensation runs after last retry failure |
| `throwOnError: false` passed to `.run()` | Errors collected in `result.errors`; no exception thrown |
| Nested `createWorkflow` (sub-workflow) | Sub-workflow compensation integrates into parent stack |

---

## 6. Compatibility Matrix

| SDK Version | Orchestration version | Node.js |
|---|---|---|
| 2.15.4 | 2.15.4 | ≥ 20 LTS |
| 2.14.x | 2.14.x | ≥ 18 LTS |

---

## 7. Migration Notes (2.14 → 2.15)

| Area | Change |
|---|---|
| `createHook` | Now strongly typed with hook payload generic |
| `when().then()` | `then` callback return type correctly inferred as `WorkflowData<T> \| undefined` |
| `StepConfig.timeout` | New option; previously only configurable via orchestration engine directly |
| `ReturnWorkflow.runAsStep` | New method for calling a workflow as a step inside another workflow |
