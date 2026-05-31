# arc42 Architecture Document — `@medusajs/core-flows`

**Version:** 2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/core-flows` must:

- Provide compensable, production-ready workflows for all major commerce operations.
- Be extensible — external code MUST be able to add behaviour without modifying package source.
- Remain decoupled from individual module implementations.
- Execute deterministically with full rollback capability on failure.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Reliability** | A payment failure during checkout rolls back inventory reservations automatically. |
| 2 | **Extensibility** | A plugin adds a loyalty-points step to `createOrderWorkflow` via a hook, with no fork required. |
| 3 | **Testability** | Each step is testable in isolation with a mocked container. |
| 4 | **Observability** | Workflow execution state is traceable via the orchestration engine's transaction log. |

### 1.3 Stakeholders

| Role | Expectation |
|---|---|
| Medusa Application Developer | Copy-paste starting points; easy hook integration |
| Plugin Author | Stable hook API; no breaking changes between patch versions |
| Platform Engineer | Predictable compensation; no partial state after failures |
| Commerce Manager | Correct business logic for checkout, returns, exchanges |

---

## 2. Architecture Constraints

- Steps MUST NOT import directly from sibling module packages (use container resolution only).
- Compensation functions MUST be idempotent.
- Workflow IDs MUST be globally unique strings (they are used as orchestration transaction types).
- All cross-module reads MUST go through `useQueryGraphStep`.

---

## 3. System Context

```
┌────────────────────────────────────────────────────────────┐
│                     Medusa Application                     │
│                                                            │
│  API Route / Job / Subscriber                              │
│       │                                                    │
│       │  workflow(scope).run({ input })                    │
│       ▼                                                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │           @medusajs/core-flows                      │  │
│  │                                                     │  │
│  │  createWorkflow() ──► step1 → step2 → step3 → hook  │  │
│  │                           │                         │  │
│  │                    (on failure)                      │  │
│  │                    step3.compensate()                │  │
│  │                    step2.compensate()                │  │
│  │                    step1.compensate()                │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │  container.resolve(Modules.X)        │
│                     ▼                                      │
│            Commerce Module Services                        │
│         (product, order, cart, payment…)                   │
└────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Problem | Solution |
|---|---|
| Module coupling | All module access via DI container, not direct imports |
| Partial failure recovery | Every step declares a compensation function |
| Extensibility without fork | `createHook()` injection points; handlers registered externally |
| Cross-module reads | `useQueryGraphStep` + Query engine |
| Parallel efficiency | `parallelize()` for independent steps |

---

## 5. Building Blocks (Level 1)

```
@medusajs/core-flows
├── cart/       — 12 workflows, 15+ steps
├── order/      — 20+ workflows (create, cancel, return, exchange, claim…)
├── product/    — 8 workflows (CRUD + import/export)
├── payment/    — 5 workflows
├── fulfillment/— 4 workflows
├── customer/   — 3 workflows
├── auth/       — 2 workflows
├── pricing/    — 4 workflows
├── promotion/  — 5 workflows
├── inventory/  — shared steps (confirm, reserve, restore)
├── notification/— 1 workflow (send-notifications)
└── index.ts    — barrel re-exports
```

---

## 6. Building Blocks (Level 2) — `completeCartWorkflow`

```
completeCartWorkflow
│
├── getCartWorkflow [sub-workflow]
│     └── useQueryGraphStep (cart + items + variants + inventory)
│
├── confirmInventoryStep
│     └── inventoryModule.confirmInventory()
│
├── reserveInventoryStep
│     └── inventoryModule.createReservationItems()
│     └── compensate: deleteReservationItemsStep
│
├── authorizePaymentSessionWorkflow [sub-workflow]
│     └── paymentModule.authorizePaymentSession()
│     └── compensate: cancelPaymentStep
│
├── createOrderWorkflow [sub-workflow]
│     └── orderModule.createOrders()
│
├── createFulfillmentsWorkflow [sub-workflow]
│     └── fulfillmentModule.createFulfillments()
│
└── Hook: cartCompleted { order }
```

---

## 7. Runtime View

### 7.1 Happy-Path: Cart Checkout

```
Client: POST /store/carts/:id/complete
  → completeCartWorkflow(scope).run({ input: { id } })
      1. Load cart with full graph              [~5ms]
      2. Confirm inventory (SELECT FOR UPDATE)  [~3ms]
      3. Reserve inventory                      [~4ms]
      4. Authorize payment                      [~200ms — provider round-trip]
      5. Create order record                    [~8ms]
      6. Create fulfillment records             [~6ms]
      7. Fire cartCompleted hook               [async]
  ← 201 { order }
```

### 7.2 Failure Path: Payment Authorization Fails

```
      1. Load cart              ✓
      2. Confirm inventory      ✓
      3. Reserve inventory      ✓
      4. Authorize payment      ✗ (card declined)
          → compensate step 3: delete reservations
          → compensate step 2: no-op (read-only)
          → compensate step 1: no-op (read-only)
  ← 402 { type: "payment_authorization_error" }
```

---

## 8. Deployment View

`@medusajs/core-flows` is a pure library — it has no runtime service of its own. It executes within the same Node.js process as the Medusa application, using the caller's DI container.

For distributed workflow execution (e.g., long-running async workflows):

- The `@medusajs/orchestration` engine can persist workflow state to the database.
- Workflows resume from their last completed step after a process restart.
- The `idempotency_key` passed to `.run()` ensures exactly-once semantics.

---

## 9. Cross-Cutting Concerns

### 9.1 Idempotency

Workflows accept an optional `idempotencyKey` to prevent duplicate execution on retries. The orchestration engine deduplicates by key.

### 9.2 Compensation Logging

Failed compensations are logged at `ERROR` level with the step name, workflow ID, and error message. They do NOT silently swallow errors.

### 9.3 Workflow Versioning

Workflow IDs are stable strings. Adding a new step to an existing workflow is a non-breaking change as long as the step's compensation function is idempotent.

---

## 10. Architecture Decisions

### ADR-001: No Direct Module Imports

**Context:** Steps could import module service interfaces for type safety.  
**Decision:** Use `container.resolve(Modules.X)` exclusively.  
**Consequences:** Loose coupling; module implementations can change without updating core-flows. Trade-off: slightly less type inference at the call site.

### ADR-002: Sub-workflows for Reusable Sequences

**Context:** `completeCartWorkflow` and `createOrderWorkflow` share several steps.  
**Decision:** Extract shared sequences into their own workflows and call them as sub-workflows via `runAsStep()`.  
**Consequences:** Reuse without copy-paste; sub-workflow compensation runs as part of parent's compensation chain.

### ADR-003: Hooks Are Post-Commit Only

**Context:** Should hooks run inside or outside the main transaction?  
**Decision:** Hooks execute after the workflow's last step, outside the transaction boundary.  
**Consequences:** Hook failures do not roll back business data; hooks are for side-effects (notifications, search index updates) not for critical state changes.
