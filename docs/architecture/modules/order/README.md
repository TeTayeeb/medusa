# Order Module — `@medusajs/order`

> Medusa v2.15.4 · Module Reference

## Purpose & Domain Responsibility

The Order Module is the **transactional record** of commerce. It captures the complete lifecycle of a customer purchase from initial creation through fulfillment, returns, exchanges, and claims. It is the single source of truth for what was ordered, at what price, by whom, and in what state.

The module implements a **versioned order change system**: rather than mutating orders in place, all changes (returns, exchanges, edits) are tracked as `OrderChange` + `OrderChangeAction` records, enabling full audit trail and rollback semantics.

---

## Key Entities

| Entity | Prefix | Description |
|---|---|---|
| `Order` | `order_` | Root order record. Carries status, currency, customer/region references, version counter. |
| `OrderItem` | — | A line item snapshot on the order (product variant, quantity, unit price, adjustments). |
| `OrderLineItem` | — | The detailed line item including tax lines and adjustments. |
| `OrderShipping` / `OrderShippingMethod` | — | Shipping method(s) selected at checkout, with tax lines and adjustments. |
| `OrderTransaction` | — | Financial transaction record (payment capture, refund, etc.) tied to the order. |
| `OrderChange` | — | A proposed or confirmed modification (return, exchange, edit, claim). Has status `pending → confirmed → declined → canceled`. |
| `OrderChangeAction` | — | Individual action within a change (add item, remove item, ship item, receive return). |
| `Return` | `return_` | Return request with associated items and shipping. |
| `ReturnItem` | — | Individual item within a return, with reason and quantity. |
| `ReturnReason` | `rr_` | Catalogue of standard return reasons. |
| `OrderClaim` | `claim_` | A defect / dispute claim against the order. Contains claim items and images. |
| `OrderExchange` | `exchange_` | Swap of items: return old + ship new. |
| `OrderSummary` | — | Computed financial summary snapshot (per order version). |
| `OrderCreditLine` | — | Store-credit line applied to the order. |
| `OrderAddress` | — | Shipping / billing address snapshot on the order. |

---

## Key Service Methods

```ts
// Reads
listOrders(filters, config, context): Promise<OrderDTO[]>
retrieveOrder(id, config, context): Promise<OrderDTO>
listReturns(filters, config, context): Promise<ReturnDTO[]>

// Create
createOrders(data[], context): Promise<OrderDTO[]>

// Update / State
updateOrders(selector, data, context): Promise<OrderDTO[]>
cancelOrder(id, context): Promise<OrderDTO>
completeOrder(id, context): Promise<OrderDTO>

// Order Changes
createOrderChange(data, context): Promise<OrderChangeDTO>
confirmOrderChange(id, context): Promise<OrderChangeDTO>
declineOrderChange(id, context): Promise<OrderChangeDTO>
cancelOrderChange(id, context): Promise<OrderChangeDTO>
applyOrderChanges(orderId, context): Promise<void>

// Returns
createReturns(data[], context): Promise<ReturnDTO[]>
receiveReturn(returnId, items, context): Promise<ReturnDTO>
cancelReturn(returnId, context): Promise<ReturnDTO>

// Financial
createOrderTransaction(data, context): Promise<OrderTransactionDTO>
```

---

## Module Dependencies

| Dependency | Direction | Reason |
|---|---|---|
| `@medusajs/payment` | Outbound (via workflow) | Capture / refund payments on order lifecycle events |
| `@medusajs/fulfillment` | Outbound (via workflow) | Create / cancel fulfillments |
| `@medusajs/inventory` | Outbound (via workflow) | Adjust inventory on order confirm / return |
| `@medusajs/product` | Reference only | `product_id`, `variant_id` stored as strings |
| `@medusajs/event-bus` | Optional, outbound | Emits order lifecycle events |

The Order module itself is stateless w.r.t. other modules — it stores IDs as foreign keys but never joins to external modules directly.

---

## Key Workflows

| Workflow | Description |
|---|---|
| `createOrderWorkflow` | Converts a completed cart into an order; creates transactions |
| `cancelOrderWorkflow` | Cancels order; triggers fulfillment cancel + payment refund compensations |
| `completeOrderWorkflow` | Marks order as complete; finalises financial summary |
| `createReturnWorkflow` | Initiates a return request with item list and reason |
| `receiveReturnWorkflow` | Marks return items as received; adjusts inventory |
| `cancelReturnWorkflow` | Cancels a pending return |
| `createExchangeWorkflow` | Initiates exchange (return old items + create new outbound) |
| `createClaimWorkflow` | Initiates a warranty/defect claim |
| `createOrderEditWorkflow` | Opens an order edit change; adds/removes/modifies items |
| `confirmOrderEditWorkflow` | Confirms the edit; applies changes to order version |

---

## Admin API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/orders` | List orders with filters |
| `GET` | `/admin/orders/:id` | Retrieve order with full detail |
| `POST` | `/admin/orders/:id/cancel` | Cancel order |
| `POST` | `/admin/orders/:id/complete` | Complete order |
| `GET` | `/admin/returns` | List returns |
| `POST` | `/admin/returns` | Create return |
| `POST` | `/admin/returns/:id/receive` | Receive return items |
| `POST` | `/admin/returns/:id/cancel` | Cancel return |
| `GET` | `/admin/exchanges` | List exchanges |
| `POST` | `/admin/exchanges` | Create exchange |
| `GET` | `/admin/claims` | List claims |
| `POST` | `/admin/claims` | Create claim |
| `GET` | `/admin/order-edits` | List order edits |
| `POST` | `/admin/order-edits` | Create order edit |

---

## State Machine

```
pending → processing → shipped → delivered
    │           │           │
    └───────────┴───────────┴──→ cancelled
                                      │
                                   returned
```

`OrderChange` states: `pending → confirmed | declined | cancelled`

---

## Configuration Options

```ts
// medusa-config.ts
{
  resolve: "@medusajs/order",
  options: {
    database: { clientUrl: "postgresql://..." }
  }
}
```

---

## Extension Points

| Extension | How |
|---|---|
| Custom order fields | `additional_data` pattern; hook into `createOrderWorkflow` |
| Order events | Subscribe to `order.placed`, `order.cancelled`, `order.completed`, `order.return_requested` |
| Custom return reasons | `createReturnReason()` API |
| Order change hooks | `orderCreated`, `orderCancelled`, `returnCreated` workflow hooks |
| Fulfillment providers | Via `@medusajs/fulfillment` module |
