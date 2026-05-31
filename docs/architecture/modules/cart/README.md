# Cart Module — `@medusajs/cart`

> Medusa v2.15.4 · Module Reference

## Purpose & Domain Responsibility

The Cart Module manages the **pre-purchase shopping experience**. It holds the mutable, transient state of a customer's intent to buy: items they have selected, quantities, shipping address, billing address, shipping method choices, applied discounts, and computed totals.

A cart is the **staging area** before an order is created. Once `completeCartWorkflow` runs, the cart is snapshotted into an immutable `Order` and the cart is marked completed.

---

## Key Entities

| Entity | Prefix | Description |
|---|---|---|
| `Cart` | `cart_` | Root shopping cart. Carries region, customer, sales channel, currency, addresses, and all computed total fields. |
| `LineItem` | — | A single product variant selection in the cart with quantity, unit price, and thumbnail snapshot. |
| `LineItemAdjustment` | — | A promotional or manual discount adjustment applied to a specific line item. Carries `promotion_id` reference. |
| `LineItemTaxLine` | — | A tax line applied to a specific line item. Carries `tax_rate_id` and rate value. |
| `ShippingMethod` | — | A shipping option selected for the cart. Carries amount and shipping option reference. |
| `ShippingMethodAdjustment` | — | Discount adjustment on a shipping method (e.g. free shipping promotion). |
| `ShippingMethodTaxLine` | — | Tax line applied to a shipping method. |
| `Address` | — | Shipping or billing address. Reused for both; linked via `shipping_address_id` / `billing_address_id`. |
| `CreditLine` | — | A store-credit or gift-card line applied to the cart. |

---

## Key Service Methods

```ts
// Cart lifecycle
createCarts(data[], context): Promise<CartDTO[]>
retrieveCart(id, config, context): Promise<CartDTO>
updateCarts(selector, data, context): Promise<CartDTO[]>
deleteCart(id, context): Promise<void>

// Line items
addLineItems(data[], context): Promise<LineItemDTO[]>
updateLineItems(selector, data, context): Promise<LineItemDTO[]>
deleteLineItems(ids[], context): Promise<void>

// Adjustments (discounts/promotions)
addLineItemAdjustments(data[], context): Promise<LineItemAdjustmentDTO[]>
setLineItemAdjustments(cartId, data[], context): Promise<LineItemAdjustmentDTO[]>
removeLineItemAdjustments(ids[], context): Promise<void>

// Shipping
addShippingMethods(data[], context): Promise<ShippingMethodDTO[]>
removeShippingMethods(ids[], context): Promise<void>
addShippingMethodAdjustments(data[], context): Promise<ShippingMethodAdjustmentDTO[]>
setShippingMethodAdjustments(cartId, data[], context): Promise<ShippingMethodAdjustmentDTO[]>

// Tax
addLineItemTaxLines(data[], context): Promise<LineItemTaxLineDTO[]>
addShippingMethodTaxLines(data[], context): Promise<ShippingMethodTaxLineDTO[]>

// Totals (computed)
// Cart totals are computed fields on the Cart entity itself
```

---

## Module Dependencies

| Dependency | Direction | Reason |
|---|---|---|
| `@medusajs/product` | Reference only | `variant_id`, `product_id` stored on line items (no join) |
| `@medusajs/pricing` | Outbound (via workflow) | Retrieve variant prices when adding line items |
| `@medusajs/promotion` | Outbound (via workflow) | Compute adjustments after cart changes |
| `@medusajs/tax` | Outbound (via workflow) | Compute tax lines per item and shipping method |
| `@medusajs/fulfillment` | Outbound (via workflow) | List available shipping options |
| `@medusajs/payment` | Outbound (via workflow) | Create `PaymentCollection` on cart completion |
| `@medusajs/order` | Outbound (via workflow) | Create order from completed cart |

The Cart module itself stores only IDs; all cross-module enrichment happens at the workflow level.

---

## Key Workflows

| Workflow | Description |
|---|---|
| `createCartWorkflow` | Creates a cart; resolves region/sales-channel; sets currency |
| `addToCartWorkflow` | Adds a line item; fetches variant price; recalculates totals |
| `updateLineItemInCartWorkflow` | Updates quantity or options; reprices and recalculates |
| `deleteLineItemsWorkflow` | Removes items; recalculates promotions and totals |
| `updateCartWorkflow` | Updates cart-level fields (address, email, region, coupon code) |
| `addShippingMethodToCartWorkflow` | Validates and adds a shipping method; reprices |
| `applyPromotionsWorkflow` | Re-evaluates promotion adjustments on all items and shipping |
| `refreshPaymentCollectionWorkflow` | Syncs payment collection amount after cart changes |
| `completeCartWorkflow` | Authorises payment; creates order; marks cart completed |

---

## Store API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/store/carts` | Create cart |
| `GET` | `/store/carts/:id` | Retrieve cart with totals |
| `POST` | `/store/carts/:id` | Update cart (address, email, region, coupon) |
| `POST` | `/store/carts/:id/line-items` | Add line item |
| `POST` | `/store/carts/:id/line-items/:item_id` | Update line item quantity |
| `DELETE` | `/store/carts/:id/line-items/:item_id` | Remove line item |
| `POST` | `/store/carts/:id/shipping-methods` | Add shipping method |
| `POST` | `/store/carts/:id/complete` | Complete cart (checkout) |
| `POST` | `/store/payment-collections` | Create payment collection for cart |
| `POST` | `/store/payment-collections/:id/payment-sessions` | Initiate payment session |

---

## Configuration Options

```ts
// medusa-config.ts
{
  resolve: "@medusajs/cart",
  options: {
    database: { clientUrl: "postgresql://..." }
  }
}
```

---

## Extension Points

| Extension | How |
|---|---|
| Cart events | `cart.created`, `cart.updated` via Event Bus |
| Custom line item fields | `additional_data` in `addToCartWorkflow` hooks |
| Custom promotion logic | Plug into `applyPromotionsWorkflow` hooks |
| Custom tax calculation | Override tax step in `addToCartWorkflow` |
| Workflow hooks | `cartCreated`, `lineItemsAdded`, `shippingMethodsAdded` |
