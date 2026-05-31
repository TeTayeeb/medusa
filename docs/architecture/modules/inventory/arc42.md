# arc42 Architecture Document — Inventory Module

**Module:** `@medusajs/inventory`  
**Version:** Medusa v2.15.4  
**arc42 Template Version:** 8.x

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Inventory module must:
- Track on-hand stock quantities per (item, location) pair.
- Provide atomic reservation of stock during the checkout lifecycle.
- Compute `available_quantity` as `stocked_quantity - reserved_quantity` at query time.
- Support backorder flags for pre-sell scenarios.
- Integrate with product variants (via Module Links) and stock locations (via `location_id` soft reference).

### 1.2 Quality Goals
| Priority | Quality Goal | Motivation |
|---|---|---|
| 1 | **Atomicity** | Reservations must never oversell without explicit backorder permission |
| 2 | **Precision** | Quantity arithmetic must be exact — no floating-point drift |
| 3 | **Performance** | Available-quantity lookups happen on every cart add and checkout confirm |
| 4 | **Auditability** | Reservation history must be traceable via `line_item_id` |

---

## 2. Architecture Constraints

- Quantities must use `BigNumber` (NUMERIC + JSONB) — never JavaScript `number` for arithmetic.
- No direct FK to `stock_location` — `location_id` is a plain text field (soft reference).
- No direct FK to product variants — linked via Module Link (`product_variant_inventory_item`).
- Reservation creation must be atomic with the availability check (transaction-scoped).

---

## 3. System Scope and Context

```
┌───────────────────────────────────────────────────────────┐
│                     Medusa Application                    │
│                                                           │
│  [Product Module] ──link── [Inventory Module]             │
│  (variant→inventory_item)         │                       │
│                          [Stock-Location Module]          │
│                          (location_id soft ref)           │
│                                                           │
│  [Cart/Order Module] ──────────→ reservations             │
│  (line_item_id)          [Fulfillment Module]             │
│                          (reads availability)             │
└───────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Decision | Rationale |
|---|---|
| Three-entity model | InventoryItem (what), InventoryLevel (where + how many), ReservationItem (temporarily held). Each has a distinct lifecycle. |
| Computed `available_quantity` | Not stored — computed from `stocked_quantity - reserved_quantity` to guarantee consistency without update races. |
| `InventoryLevelService` specialisation | Standard CRUD is insufficient for `adjustInventory` (SELECT FOR UPDATE) and aggregated quantity queries. |
| `allow_backorder` per reservation | Oversell is opt-in at the reservation level, supporting pre-orders without changing stock amounts. |

---

## 5. Building Block View

### Level 1 — Module Structure

```
@medusajs/inventory
├── models/
│   ├── inventory-item.ts
│   ├── inventory-level.ts
│   └── reservation-item.ts
├── repositories/
│   └── inventory-level.ts       ← Custom quantity aggregation
├── services/
│   ├── inventory-module.ts      ← IInventoryService implementation
│   └── inventory-level.ts       ← InventoryLevelService (specialised)
├── utils/
│   └── apply-decorators.ts      ← Re-applies @InjectManager to computed methods
├── joiner-config.ts
└── index.ts
```

### Level 2 — InventoryLevelService

```
InventoryLevelService extends MedusaInternalService<InventoryLevel>
  ├── adjustInventory(itemId, locationId, adjustment, ctx)
  │     → SELECT FOR UPDATE → UPDATE stocked_quantity += adjustment
  ├── getAvailableQuantity(itemId, locationIds[], ctx)
  │     → SELECT SUM(stocked_quantity - reserved_quantity) GROUP BY inventory_item_id
  ├── getStockedQuantity(itemId, locationIds[], ctx)
  └── getReservedQuantity(itemId, locationIds[], ctx)
```

---

## 6. Runtime View

### 6.1 Add to Cart (Reservation Creation)

```
POST /store/carts/:id/line-items
  → addToCartWorkflow
      → getVariantPriceSetsStep
      → createLineItemsStep (Cart Module)
      → createReservationsStep (Inventory Module)
          → confirmInventory(itemId, [locationId], qty)
             → if available < qty AND !allow_backorder → throw NOT_ALLOWED
          → createReservationItem(line_item_id, itemId, locationId, qty)
```

### 6.2 Order Completion (Stock Deduction)

```
POST /admin/orders/:id/fulfillments
  → createFulfillmentWorkflow
      → adjustInventoryAndReservationsStep
          → deleteReservationItems(reservation_ids)
          → adjustInventory(itemId, locationId, -fulfilled_qty)
```

### 6.3 Order Cancellation

```
POST /admin/orders/:id/cancel
  → cancelOrderWorkflow
      → deleteReservationsByLineItemsStep
          → deleteReservationItemsByLineItem(line_item_ids)
          → (stocked_quantity unchanged — no stock was deducted)
```

---

## 7. Deployment View

The module runs in-process. The `adjustInventory` operation uses PostgreSQL row-level locking (`SELECT FOR UPDATE`) to prevent concurrent oversell. For high-concurrency environments, a distributed lock (via `@medusajs/locking`) wraps the reserve+check critical section at the workflow level.

---

## 8. Cross-Cutting Concepts

### 8.1 BigNumber Dual-Storage
See Pricing module arc42 §8.1. Same pattern applies. `BigNumber(stocked).subtract(BigNumber(reserved))` is performed in application code, not SQL, to maintain precision.

### 8.2 Soft-Delete Cascade
Deleting an `InventoryItem` cascades to its `InventoryLevel` and `ReservationItem` records. This prevents orphaned reservations holding ghost stock.

### 8.3 Location Soft Reference
`location_id` is stored as a plain `TEXT` column with no DB FK to `stock_location`. This allows inventory records to survive stock location deletion and be manually reconciled.

---

## 9. Architecture Decisions

### ADR-01: No DB FK to StockLocation
**Status:** Accepted  
**Context:** A DB FK would create a hard dependency, preventing inventory data from existing without a corresponding stock location (e.g. during import or migration).  
**Decision:** `location_id` is a plain text column with an application-level convention that it references `StockLocation.id`.  
**Consequences:** No referential integrity at DB level; application must ensure locations exist before creating levels.

### ADR-02: Computed available_quantity
**Status:** Accepted  
**Context:** Storing `available_quantity` requires keeping it in sync on every reservation change.  
**Decision:** Compute `available_quantity = stocked - reserved` on read. Store `stocked_quantity` and `reserved_quantity` separately.  
**Consequences:** No sync bugs; slightly more work on read (simple subtraction).

### ADR-03: InventoryLevelService Specialisation
**Status:** Accepted  
**Context:** Standard `MedusaInternalService` cannot safely implement `SELECT FOR UPDATE` atomic adjustments.  
**Decision:** Extend the base service with custom methods using raw ORM queries where needed.  
**Consequences:** More maintainable than raw SQL, while still providing the necessary concurrency control.

---

## 10. Risks and Technical Debt

| Risk | Severity | Mitigation |
|---|---|---|
| Concurrent `adjustInventory` calls | High | `SELECT FOR UPDATE` row lock in PostgreSQL |
| Orphaned reservations on order failure | Medium | Compensation steps in workflows delete reservations on rollback |
| Missing inventory levels on variant link | Low | `ensureInventoryLevels` auto-creates missing levels |
