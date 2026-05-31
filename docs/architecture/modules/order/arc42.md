# arc42 Architecture Documentation — Order Module

> Medusa v2.15.4 | Focused sections: 1, 3, 5, 6, 8, 9

---

## Section 1 — Introduction and Goals

### 1.1 Purpose

The Order Module is the **transactional heart** of Medusa. It records every commercial event — purchase, fulfilment, return, exchange, claim — and maintains a complete, auditable history of each order through its lifecycle.

**Stakeholders:**
- **Merchants / Ops teams** — process orders, handle returns, create refunds
- **Customers** — view order status, request returns
- **Fulfillment & Payment modules** — receive hooks to act on order state changes
- **Finance/Accounting systems** — consume `OrderTransaction` and `OrderSummary` data

### 1.2 Quality Goals

| Priority | Goal | Scenario |
|---|---|---|
| 1 | **Data Integrity** | Order totals must always match sum of items + shipping; no rounding errors |
| 2 | **Auditability** | Every mutation to an order must be traceable via `OrderChange` + `OrderChangeAction` |
| 3 | **Consistency** | State transitions must be valid; invalid transitions must be rejected atomically |
| 4 | **Extensibility** | Returns, exchanges, claims must be independently evolvable without touching `Order` |

---

## Section 3 — Building Block View

### Level 1 — Context

```
┌──────────────────────────────────────────────────┐
│                  Medusa Application               │
│                                                   │
│  Admin API ──▶  createOrderWorkflow               │
│  Store API ──▶  createReturnWorkflow              │
│                      │                            │
│                 OrderModuleService                │
│                 (IOrderModuleService)             │
│                      │                            │
│                 ┌────┴────────────────────┐       │
│                 │  OrderService (CRUD)    │       │
│                 │  BundledActions         │       │
│                 │  applyChangesToOrder()  │       │
│                 └────────────────────────┘       │
│                      │                            │
│              PostgreSQL (MikroORM)                │
│  Event Bus ◀── order.placed / order.cancelled ... │
└──────────────────────────────────────────────────┘
```

### Level 2 — Key Components

| Component | Responsibility |
|---|---|
| `OrderModuleService` | Public API surface; orchestrates changes, returns, exchanges |
| `OrderService` | Low-level CRUD for `Order` entity; status guard |
| `BundledActions` | Stateless handlers for each `ChangeActionType` (ADD_ITEM, SHIP_ITEM, RECEIVE_RETURN…) |
| `calculateOrderChange()` | Computes the financial delta of a proposed change set |
| `applyChangesToOrder()` | Iterates `OrderChangeAction[]` and dispatches to handlers |
| `formatOrder()` / `decorateCartTotals()` | Recalculates all totals using `BigNumber` math |

---

## Section 5 — Runtime View: Create Return

```
Merchant → POST /admin/returns
    │
    ├─ createReturnWorkflow.run({ order_id, items, reason })
    │       │
    │       ├─ [step] createReturnStep
    │       │      └─ OrderModuleService.createReturns([data])
    │       │              ├─ validate order is in returnable state
    │       │              ├─ INSERT Return
    │       │              └─ INSERT ReturnItem(s)
    │       │
    │       ├─ [step] createOrderChangeStep
    │       │      └─ OrderModuleService.createOrderChange({
    │       │              order_id, actions: [RETURN_ITEM × n]
    │       │         })
    │       │
    │       ├─ [step] createShippingForReturn (optional)
    │       │
    │       └─ [step] confirmOrderChangeStep
    │              └─ OrderModuleService.confirmOrderChange(changeId)
    │                      └─ applyChangesToOrder() → BundledActions.ReceiveReturn
    │
    └─ emit order.return_requested
```

---

## Section 6 — Runtime View: Cancel Order

```
Merchant → POST /admin/orders/:id/cancel
    │
    ├─ cancelOrderWorkflow.run({ id })
    │       │
    │       ├─ [step] cancelFulfillmentsStep    (Fulfillment module)
    │       ├─ [step] cancelPaymentsStep        (Payment module)
    │       ├─ [step] cancelOrderStep
    │       │      └─ OrderModuleService.cancelOrder(id)
    │       │              ├─ guard: status must be pending or processing
    │       │              ├─ UPDATE order.status = 'cancelled'
    │       │              └─ UPDATE order.canceled_at = NOW()
    │       └─ emit order.cancelled
    │
    └─ Response: { order: OrderDTO }

Compensation (if cancelOrder fails):
    ← uncancelPaymentsStep
    ← restoreFulfillmentsStep
```

---

## Section 8 — Crosscutting Concepts

### BigNumber / Monetary Precision

All amounts use `BigNumber` (backed by `decimal.js`). Raw values are stored as `{ value: string, precision: number }` JSON. Computed columns are `NUMERIC(20, 4)`. The `createRawPropertiesFromBigNumber()` utility populates both at write time.

### Versioned Order Snapshots

`Order.version` increments on every confirmed `OrderChange`. `OrderSummary` stores a full financial snapshot per version. This is an append-only audit log pattern — no in-place mutation of historical summaries.

### Change Action Dispatch

`ChangeActionType` is a closed enum. `applyChangesToOrder()` uses a `Map<ChangeActionType, BundledAction>` lookup table. Adding a new action type requires only: (a) extending the enum, (b) registering a new handler — no changes to the orchestrator.

### Soft Delete

All entities are soft-deleted. `Order.deleted_at` has a dedicated partial index for query performance on the common case (active orders).

---

## Section 9 — Architecture Decisions

### ADR-ORD-001: Order Change as Two-Phase Commit

**Decision**: Order mutations follow create-change → confirm-change flow.  
**Rationale**: Prevents partial updates; enables order edits to be drafted and reviewed before applying.  
**Consequence**: All mutation workflows must go through `OrderChange` — direct field edits are not permitted.

### ADR-ORD-002: BigNumber for All Monetary Fields

**Decision**: All money values use `BigNumber` with raw storage.  
**Rationale**: IEEE 754 double-precision floats are unsafe for financial arithmetic.  
**Consequence**: Extra storage column per monetary field; all sum operations must use `MathBN.*` functions.

### ADR-ORD-003: Returns / Exchanges / Claims as Separate Entities

**Decision**: Returns, Exchanges, and Claims are first-class entities rather than sub-states of Order.  
**Rationale**: Each has distinct lifecycles, independent item lists, and separate financial implications.  
**Consequence**: Higher model complexity; joins required when computing full order financial picture.

### ADR-ORD-004: Immutable OrderSummary per Version

**Decision**: `OrderSummary` is written once per version and never updated.  
**Rationale**: Financial summaries must be reproducible for accounting and dispute resolution.  
**Consequence**: Storage grows linearly with order mutations; old summaries are retained indefinitely.
