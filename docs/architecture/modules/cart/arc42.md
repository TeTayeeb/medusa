# arc42 Architecture Documentation — Cart Module

> Medusa v2.15.4 | Focused sections: 1, 3, 5, 6, 8, 9

---

## Section 1 — Introduction and Goals

### 1.1 Purpose

The Cart Module manages the **mutable pre-purchase state** for a customer session. It stores what the customer intends to buy, how they want it shipped, and any promotions or credits they have applied. The cart is a **temporary workspace** — it exists only until it is either completed (→ Order) or abandoned.

**Stakeholders:**
- **Customers** — add/remove items, apply coupons, select shipping
- **Storefront developers** — primary consumers of Cart APIs
- **Promotion Module** — writes adjustments into the cart
- **Tax Module** — writes tax lines into the cart
- **Payment Module** — creates payment collections referencing the cart

### 1.2 Quality Goals

| Priority | Goal | Scenario |
|---|---|---|
| 1 | **Total Accuracy** | Cart totals must reflect all adjustments and tax lines with no rounding drift |
| 2 | **Responsiveness** | `retrieveCart` with totals must respond < 100 ms for typical carts (≤ 50 items) |
| 3 | **Idempotency** | Applying the same promotion twice must produce identical state |
| 4 | **Isolation** | Cart changes must not affect other users' carts |

---

## Section 3 — Building Block View

### Level 1 — Context

```
┌──────────────────────────────────────────────────────┐
│                   Medusa Application                  │
│                                                       │
│  Store API ──▶ addToCartWorkflow                     │
│                     │                                 │
│  ┌──────────────────▼──────────────────┐             │
│  │         CartModuleService            │             │
│  │         (ICartModuleService)         │             │
│  └──────────────────┬──────────────────┘             │
│                     │                                 │
│  ┌──────────────────▼──────────────────┐             │
│  │       PostgreSQL (MikroORM)          │             │
│  └─────────────────────────────────────┘             │
│                                                       │
│  External modules (called BY workflows, not cart):   │
│  Pricing ──▶ resolve variant prices                  │
│  Promotion ──▶ compute adjustments                   │
│  Tax ──▶ compute tax lines                           │
│  Fulfillment ──▶ list shipping options               │
└──────────────────────────────────────────────────────┘
```

### Level 2 — Key Components

| Component | Responsibility |
|---|---|
| `CartModuleService` | All cart CRUD; adjustment set management; normalisation |
| `decorateCartTotals()` | Recomputes all 25+ total fields from raw items + adjustments |
| `normalizeCurrencyCode()` | Ensures lowercase ISO 4217 on every write |
| `createRawPropertiesFromBigNumber()` | Populates both `raw_` and computed BigNumber columns |

---

## Section 5 — Runtime View: Add to Cart

```
Customer → POST /store/carts/:id/line-items { variant_id, quantity }
    │
    ├─ addToCartWorkflow.run({ cartId, items: [{ variant_id, quantity }] })
    │       │
    │       ├─ [step] validateVariantStep
    │       │      └─ ProductModule.listProductVariants({ id: variant_id })
    │       │
    │       ├─ [step] getVariantPricingStep
    │       │      └─ PricingModule.calculateVariantPrice(variant_id, cartContext)
    │       │
    │       ├─ [step] addLineItemsStep
    │       │      └─ CartModuleService.addLineItems([{
    │       │              cart_id, variant_id, quantity,
    │       │              unit_price (from pricing), title, thumbnail
    │       │         }])
    │       │
    │       ├─ [step] refreshAdjustmentsStep
    │       │      └─ PromotionModule.computeActions(cart)
    │       │      └─ CartModuleService.setLineItemAdjustments(cartId, actions)
    │       │
    │       ├─ [step] refreshTaxLinesStep
    │       │      └─ TaxModule.getTaxLines(cart)
    │       │      └─ CartModuleService.addLineItemTaxLines(lines)
    │       │
    │       └─ [hook] lineItemsAdded
    │
    └─ Response: { cart: CartDTO } (with recomputed totals)
```

---

## Section 6 — Runtime View: Complete Cart (Checkout)

```
Customer → POST /store/carts/:id/complete
    │
    ├─ completeCartWorkflow.run({ id })
    │       │
    │       ├─ [step] validateCartStep           # shipping address, payment session
    │       ├─ [step] authorizePaymentSessionStep
    │       │      └─ PaymentModule.authorizePaymentSession(sessionId)
    │       │
    │       ├─ [step] createOrderFromCartStep
    │       │      └─ OrderModule.createOrders([cartSnapshot])
    │       │
    │       ├─ [step] completeCartStep
    │       │      └─ CartModuleService.updateCarts({ completed_at: NOW() })
    │       │
    │       └─ emit order.placed
    │
    └─ Response: { type: "order", order: OrderDTO }
    
Compensation (if order creation fails):
    ← voidPaymentSessionStep
```

---

## Section 8 — Crosscutting Concepts

### Total Computation

`decorateCartTotals()` is a **pure function** that accepts the cart with all items, adjustments, and tax lines, and returns enriched DTOs with all totals computed. It uses `MathBN.*` (BigNumber arithmetic) to avoid floating-point errors. Totals are re-computed on every `retrieveCart` call — they are **not persisted** as canonical values (only as cached computed columns on the entity for DB-level filtering).

### Set-Semantics for Adjustments

`setLineItemAdjustments()` and `setShippingMethodAdjustments()` use **replace-all** semantics: all existing adjustments are deleted and the new set is inserted atomically in a single transaction. This is necessary because promotion engines emit a complete computed set, not a diff.

### Address Snapshotting

When `completeCartWorkflow` runs, cart addresses are **copied** to `OrderAddress` records. The cart address entity is not referenced from the order; this ensures cart address mutations after order creation do not affect order records.

### Soft Delete & `completed_at`

Completed carts are not soft-deleted immediately — they are marked with `completed_at` timestamp. Cleanup is handled by a background job. This preserves cart data for analytics.

---

## Section 9 — Architecture Decisions

### ADR-CART-001: Cart Totals as Computed Fields Only

**Decision**: All cart totals are computed at read time by `decorateCartTotals()`, not persisted as canonical DB columns.  
**Rationale**: Storing totals would require recomputation triggers on every item/adjustment write, creating write amplification.  
**Consequence**: Every `retrieveCart` invocation recomputes totals; caching is handled at API layer.

### ADR-CART-002: Module Stores Only IDs for Cross-Module References

**Decision**: Cart line items store `variant_id`, `product_id`, `product_title`, `variant_title`, `thumbnail` as **snapshot strings**, not live foreign keys.  
**Rationale**: Product catalog changes must not retroactively alter cart line item display data.  
**Consequence**: Cart line items can display stale data if a product is updated while the item is in cart.

### ADR-CART-003: No Direct Inter-Module Calls from CartModuleService

**Decision**: `CartModuleService` never calls Pricing, Promotion, Tax, or Fulfillment modules.  
**Rationale**: Keeps the module cohesive and independently testable; all orchestration lives in workflows.  
**Consequence**: All business logic (pricing, promotions, tax) must be re-applied by workflow steps after every cart mutation.

### ADR-CART-004: CreditLine Excluded from Cascade Delete

**Decision**: `CreditLine` rows are not cascade-deleted when a cart is deleted.  
**Rationale**: Prevents accidental store credit loss if a cart is cleaned up; credits must be explicitly voided.  
**Consequence**: Orphaned credit lines require explicit cleanup in cart abandonment flows.
