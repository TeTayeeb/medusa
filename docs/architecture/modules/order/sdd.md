# Software Design Document — Order Module

> Medusa v2.15.4

## 1. Module Architecture

```
packages/modules/order/src/
├── index.ts
├── joiner-config.ts
├── models/
│   ├── order.ts                    # Root order entity
│   ├── order-item.ts               # Order item (version-aware snapshot)
│   ├── line-item.ts                # Detailed line item
│   ├── line-item-adjustment.ts     # Discount/promotion adjustments
│   ├── line-item-tax-line.ts       # Per-item tax lines
│   ├── order-shipping-method.ts    # Shipping (OrderShipping pivot)
│   ├── shipping-method.ts
│   ├── shipping-method-adjustment.ts
│   ├── shipping-method-tax-line.ts
│   ├── transaction.ts              # Financial transactions
│   ├── order-change.ts             # Change set header
│   ├── order-change-action.ts      # Individual change actions
│   ├── return.ts                   # Return header
│   ├── return-item.ts              # Return item
│   ├── return-reason.ts            # Return reason catalogue
│   ├── claim.ts                    # Claim header
│   ├── claim-item.ts               # Claim item
│   ├── claim-item-image.ts         # Evidence images for claim
│   ├── exchange.ts                 # Exchange header
│   ├── exchange-item.ts            # Exchange item
│   ├── order-summary.ts            # Financial summary snapshot
│   ├── credit-line.ts              # Store-credit lines
│   └── address.ts
├── services/
│   ├── order-module-service.ts     # IOrderModuleService implementation
│   ├── order-service.ts            # Low-level CRUD for Order entity
│   ├── actions/                    # Bundled action implementations
│   │   ├── add-line-item.ts
│   │   ├── remove-line-item.ts
│   │   ├── ship-item.ts
│   │   ├── receive-return-item.ts
│   │   └── ...
│   └── index.ts
├── utils/
│   ├── apply-changes.ts            # applyChangesToOrder()
│   ├── calculate-order-change.ts   # calculateOrderChange()
│   ├── format-order.ts             # formatOrder() — compute totals
│   └── events.ts
└── migrations/
```

---

## 2. Data Model

### Entity-Relationship Overview

```
Order (1) ──── (*) OrderItem
  │              └── (*) OrderLineItem
  │                    ├── (*) LineItemAdjustment
  │                    └── (*) LineItemTaxLine
  │
  ├── (*) OrderShipping
  │         └── ShippingMethod
  │               ├── (*) ShippingMethodAdjustment
  │               └── (*) ShippingMethodTaxLine
  │
  ├── (*) OrderTransaction
  ├── (*) OrderSummary         [one per version]
  ├── (*) OrderCreditLine
  ├── (*) Return
  │         └── (*) ReturnItem
  ├── (*) OrderChange
  │         └── (*) OrderChangeAction
  ├── (*) OrderClaim
  │         └── (*) OrderClaimItem
  │               └── (*) OrderClaimItemImage
  └── (*) OrderExchange
            └── (*) OrderExchangeItem
```

### Versioning System

`Order.version` is an integer that increments with every confirmed `OrderChange`. `OrderSummary` records one financial snapshot per version. This allows historical total computation without mutating past records.

### BigNumber Storage

All monetary amounts (unit_price, subtotal, tax_total, etc.) are stored as both:
- `raw_{field}` — `JSON` column containing precise `{ value, precision }` representation
- `{field}` — computed `NUMERIC` column for querying

This ensures floating-point safety at all precision levels.

---

## 3. Service Layer Design

### Class Hierarchy

```
MedusaService<EntityMap>
    └── OrderModuleService (implements IOrderModuleService)
            ├── Auto-CRUD: list*, retrieve*, create*, update*, delete*
            ├── Custom: createOrderChange / confirmOrderChange / applyOrderChanges
            ├── Custom: calculateOrderChange()
            ├── Custom: formatOrder() — recalculate totals
            └── BundledActions: AddLineItem, RemoveLineItem, ShipItem, ...
```

### Order Change Architecture

Order mutations go through a **two-phase commit** pattern:
1. `createOrderChange()` + `addOrderChangeActions()` — creates a `pending` change set with typed actions
2. `confirmOrderChange()` — validates then calls `applyChangesToOrder()` which iterates actions and dispatches to the corresponding `BundledAction` handler
3. The order's `version` increments and a new `OrderSummary` is computed

### Dependency Injection

```ts
type InjectedDependencies = {
  baseRepository: DAL.RepositoryService
  orderService: OrderService
  orderAddressService: IMedusaInternalService
  orderLineItemService: IMedusaInternalService
  orderShippingMethodService: IMedusaInternalService
  orderTransactionService: IMedusaInternalService
  orderChangeService: IMedusaInternalService
  orderChangeActionService: IMedusaInternalService
  orderItemService: IMedusaInternalService
  orderSummaryService: IMedusaInternalService
  returnService: IMedusaInternalService
  returnItemService: IMedusaInternalService
  returnReasonService: IMedusaInternalService
  orderClaimService: IMedusaInternalService
  orderExchangeService: IMedusaInternalService
  orderCreditLineService: IMedusaInternalService
}
```

---

## 4. Repository Pattern

`OrderService` (the low-level service) extends `MedusaInternalService` and delegates to MikroORM's EntityManager. There is no custom repository; complex queries use the Query Builder via `FindConfig.relations` and `FilterableOrderProps`.

Totals are not stored as individual columns but **computed** via `formatOrder()` / `decorateCartTotals()` at read time using the `BigNumber` math library. The `OrderSummary` entity caches the result per version.

---

## 5. Events Emitted

| Event | Trigger |
|---|---|
| `order.placed` | After `createOrderWorkflow` completes |
| `order.cancelled` | After `cancelOrderWorkflow` completes |
| `order.completed` | After `completeOrderWorkflow` |
| `order.return_requested` | After `createReturnWorkflow` |
| `order.items_returned` | After `receiveReturnWorkflow` |
| `order.exchange_created` | After `createExchangeWorkflow` |
| `order.claim_created` | After `createClaimWorkflow` |
| `order.edit_confirmed` | After `confirmOrderEditWorkflow` |

All events carry `{ id: string }` minimum payload; consumers use `remoteQuery` to fetch full state.

---

## 6. Error Handling

| Scenario | Error Type | Detail |
|---|---|---|
| Order not found | `NOT_FOUND` | `"Order with id: {id} was not found"` |
| Cancel non-cancellable order | `NOT_ALLOWED` | `"Order in state {status} cannot be cancelled"` |
| Change on non-editable order | `NOT_ALLOWED` | `"Order must be pending/processing to accept changes"` |
| Duplicate change | `INVALID_DATA` | `"An active order change already exists"` |
| Return quantity exceeds ordered | `INVALID_DATA` | `"Cannot return more than ordered quantity"` |
| BigNumber precision error | `INVALID_DATA` | Invalid monetary amount format |
