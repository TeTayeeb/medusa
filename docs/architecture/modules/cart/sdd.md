# Software Design Document — Cart Module

> Medusa v2.15.4

## 1. Module Architecture

```
packages/modules/cart/src/
├── index.ts                         # Module entry, DI wiring
├── models/
│   ├── cart.ts                      # Root Cart entity
│   ├── line-item.ts                 # LineItem entity
│   ├── line-item-adjustment.ts      # Discount adjustments
│   ├── line-item-tax-line.ts        # Tax lines per item
│   ├── shipping-method.ts           # Shipping selection
│   ├── shipping-method-adjustment.ts
│   ├── shipping-method-tax-line.ts
│   ├── address.ts                   # Reusable address
│   ├── credit-line.ts               # Store credit / gift card
│   └── index.ts
├── services/
│   ├── cart-module.ts               # ICartModuleService implementation
│   └── index.ts
├── types/
│   ├── index.ts                     # Internal DTO types
│   └── ...
└── migrations/
```

The Cart module is intentionally **thin**: it stores cart structure and delegates complex pricing, promotion, tax, and shipping logic to dedicated modules via workflow steps.

---

## 2. Data Model

### Entity-Relationship Overview

```
Cart (1) ──── (*) LineItem
  │                ├── (*) LineItemAdjustment   [promotion discount]
  │                └── (*) LineItemTaxLine      [tax rate applied]
  │
  ├── (*) ShippingMethod
  │         ├── (*) ShippingMethodAdjustment   [shipping discount]
  │         └── (*) ShippingMethodTaxLine      [shipping tax]
  │
  ├── (*) CreditLine                           [gift cards, store credit]
  ├── (1) Address [shipping_address_id]
  └── (1) Address [billing_address_id]
```

### Computed Total Fields

`Cart` carries **25+ computed BigNumber fields** that are recalculated by `decorateCartTotals()` whenever the cart is read. They are stored on the entity but always re-derived from the underlying items:

```
original_item_total     item_total      original_total      total
original_item_subtotal  item_subtotal   original_subtotal   subtotal
original_item_tax_total item_tax_total  original_tax_total  tax_total
                                        discount_total      discount_tax_total
shipping_total          shipping_subtotal   shipping_tax_total
original_shipping_total original_shipping_subtotal original_shipping_tax_total
gift_card_total         gift_card_tax_total
```

### Address Reuse

`Address` is stored as a separate entity referenced by both `shipping_address_id` and `billing_address_id`. On cart completion, address data is **snapshotted** onto the `Order` entity to avoid mutation of historical order data.

---

## 3. Service Layer Design

### Class Hierarchy

```
ModulesSdkUtils.MedusaService<EntityMap>
    └── CartModuleService (implements ICartModuleService)
            ├── Auto-CRUD for all 9 entities
            ├── Custom: addLineItems()       – normalises currency, generates IDs
            ├── Custom: addShippingMethods() – validates method structure
            ├── Custom: setLineItemAdjustments()  – replace-all pattern
            └── Custom: setShippingMethodAdjustments()
```

### Key Design Choices

**Set-semantics for adjustments**: `setLineItemAdjustments()` deletes all existing adjustments for a cart and inserts the new set atomically. This simplifies promotion recalculation — the promotions module emits a complete new set, not a diff.

**Currency normalisation**: `normalizeCurrencyCode()` is called on every cart create/update to ensure lowercase ISO 4217 codes.

**No direct pricing**: The cart module stores the `unit_price` as a BigNumber value passed in by the workflow. It never calls the Pricing module directly.

### Dependency Injection

```ts
type InjectedDependencies = {
  baseRepository: DAL.RepositoryService
  cartService: IMedusaInternalService<Cart>
  addressService: IMedusaInternalService<Address>
  lineItemService: IMedusaInternalService<LineItem>
  shippingMethodService: IMedusaInternalService<ShippingMethod>
  shippingMethodAdjustmentService: IMedusaInternalService<ShippingMethodAdjustment>
  lineItemAdjustmentService: IMedusaInternalService<LineItemAdjustment>
  lineItemTaxLineService: IMedusaInternalService<LineItemTaxLine>
  shippingMethodTaxLineService: IMedusaInternalService<ShippingMethodTaxLine>
}
```

---

## 4. Repository Pattern

No custom repository layer — all persistence goes through `MedusaInternalService` which wraps MikroORM's `EntityManager`. Cascade deletes handle cleanup: `Cart → items, shipping_methods, shipping_address, billing_address`.

`CreditLine` entities are intentionally **not** in the cascade chain — they must be explicitly deleted to avoid accidental credit loss.

---

## 5. Events Emitted

| Event | Trigger |
|---|---|
| `cart.created` | `createCarts()` |
| `cart.updated` | `updateCarts()` |
| `cart.completed` | `completeCartWorkflow` (sets `completed_at`) |

Cart events are minimal — the workflow hooks (`cartCreated`, `lineItemsAdded`, etc.) provide richer extension points for downstream logic.

---

## 6. Error Handling

| Scenario | Error Type | Message |
|---|---|---|
| Cart not found | `NOT_FOUND` | `"Cart with id: {id} was not found"` |
| Item not in cart | `NOT_FOUND` | `"Line item {id} not found in cart {cartId}"` |
| Cart already completed | `NOT_ALLOWED` | `"Cart {id} is already completed"` |
| Invalid currency | `INVALID_DATA` | Invalid ISO 4217 code |
| Negative quantity | `INVALID_DATA` | Quantity must be a positive integer |
| Missing shipping address on complete | `INVALID_DATA` | `"Shipping address is required to complete checkout"` |
