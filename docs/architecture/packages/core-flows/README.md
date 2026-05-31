# `@medusajs/core-flows` — Pre-built Commerce Workflows

**Version:** 2.15.4  
**License:** MIT  
**Category:** Commerce Orchestration

---

## Overview

`@medusajs/core-flows` is the official catalogue of ready-to-use Medusa workflows. It contains every workflow and step that powers the out-of-the-box Medusa API — from cart creation to order returns, from product import to payment capture.

Each workflow is composed using the `@medusajs/framework/workflows-sdk` primitives (`createWorkflow`, `createStep`, `createHook`, `parallelize`, `when`, `transform`) and can be executed independently, extended via hooks, or used as a reference when authoring custom workflows.

---

## Workflow Categories

### Cart

| Workflow | ID | Description |
|---|---|---|
| `createCartsWorkflow` | `create-carts` | Creates one or more carts with line items, shipping address, and customer |
| `addToCartWorkflow` | `add-to-cart` | Validates inventory, reserves stock, adds line items |
| `updateCartWorkflow` | `update-cart` | Updates cart fields and recalculates promotions and tax |
| `completeCartWorkflow` | `complete-cart` | Confirms inventory, charges payment, creates order |
| `addShippingMethodToCartWorkflow` | `add-shipping-method-to-cart` | Adds a shipping option and recalculates totals |
| `transferCartCustomerWorkflow` | `transfer-cart-customer` | Associates an anonymous cart with an authenticated customer |

### Order

| Workflow | ID | Description |
|---|---|---|
| `createOrderWorkflow` | `create-order` | Creates an order from a completed cart |
| `cancelOrderWorkflow` | `cancel-order` | Cancels an order, voids payment, returns inventory |
| `createReturnWorkflow` | `create-return` | Initiates a customer return with optional refund |
| `createExchangeWorkflow` | `create-exchange` | Processes a product exchange (return + new items) |
| `createClaimWorkflow` | `create-claim` | Handles warranty or damage claims |
| `requestOrderTransferWorkflow` | `request-order-transfer` | Transfers order ownership between customers |
| `archiveOrdersWorkflow` | `archive-orders` | Archives completed orders |

### Product

| Workflow | ID | Description |
|---|---|---|
| `createProductsWorkflow` | `create-products` | Creates products with variants, options, and media |
| `updateProductsWorkflow` | `update-products` | Updates product fields and re-indexes search |
| `deleteProductsWorkflow` | `delete-products` | Soft-deletes products and associated data |
| `importProductsWorkflow` | `import-products` | Bulk imports products from CSV |
| `exportProductsWorkflow` | `export-products` | Exports products to CSV |

### Payment

| Workflow | ID | Description |
|---|---|---|
| `createPaymentCollectionWorkflow` | `create-payment-collection` | Creates a payment collection for a cart/order |
| `authorizePaymentSessionWorkflow` | `authorize-payment-session` | Authorises a payment with the payment provider |
| `capturePaymentWorkflow` | `capture-payment` | Captures an authorised payment |
| `refundPaymentWorkflow` | `refund-payment` | Issues a full or partial refund |

### Fulfillment & Inventory

| Workflow | ID | Description |
|---|---|---|
| `createFulfillmentWorkflow` | `create-fulfillment` | Creates a fulfillment for order line items |
| `cancelFulfillmentWorkflow` | `cancel-fulfillment` | Cancels fulfillment and re-adjusts inventory |
| `confirmInventoryStep` | _(step)_ | Validates stock levels across locations |
| `reserveInventoryStep` | _(step)_ | Places inventory reservations |

### Customer & Auth

| Workflow | ID | Description |
|---|---|---|
| `createCustomersWorkflow` | `create-customers` | Registers new customers |
| `updateCustomersWorkflow` | `update-customers` | Updates customer profiles |
| `generateJwtTokenWorkflow` | `generate-jwt-token` | Issues a JWT for authenticated sessions |

---

## Installation

```bash
yarn add @medusajs/core-flows
```

---

## Usage Examples

### Running a Workflow from an API Route

```typescript
// src/api/store/checkout/route.ts
import { completeCartWorkflow } from "@medusajs/core-flows"
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"

export const POST = async (
  req: MedusaRequest<{ cart_id: string }>,
  res: MedusaResponse
) => {
  const { result: order } = await completeCartWorkflow(req.scope).run({
    input: { id: req.body.cart_id },
  })

  res.status(201).json({ order })
}
```

### Extending a Workflow with a Hook

```typescript
// src/workflows/hooks/order-created.ts
import { createOrderWorkflow } from "@medusajs/core-flows"

createOrderWorkflow.hooks.orderCreated(
  async ({ order }, { container }) => {
    const notificationModule = container.resolve("notificationModuleService")
    await notificationModule.createNotifications({
      to: order.email,
      template: "order-confirmation",
      data: { order_id: order.id },
    })
  }
)
```

### Importing Products in Bulk

```typescript
import { importProductsWorkflow } from "@medusajs/core-flows"

const { result } = await importProductsWorkflow(scope).run({
  input: {
    fileKey: "uploads/products-2025.csv",
  },
})

console.log(`Imported ${result.summary.toCreate} products`)
```

### Adding a Custom Step to an Existing Workflow

Because core-flows workflows are registered globally, you can wrap or compose them:

```typescript
import { createStep, createWorkflow, WorkflowResponse } from "@medusajs/framework/workflows-sdk"
import { createProductsWorkflow } from "@medusajs/core-flows"

const syncToAlgoliaStep = createStep("sync-to-algolia", async (input: { product_ids: string[] }) => {
  // push to Algolia index
})

export const createAndSyncProductsWorkflow = createWorkflow(
  "create-and-sync-products",
  (input) => {
    const { result: products } = createProductsWorkflow.runAsStep({ input })
    syncToAlgoliaStep({ product_ids: products.map((p) => p.id) })
    return new WorkflowResponse(products)
  }
)
```

---

## Architecture at a Glance

Every workflow follows the same three-layer structure:

```
workflows/<domain>/<workflow-name>.ts  ← createWorkflow() definition
steps/<domain>/<step-name>.ts          ← createStep() definitions
utils/<domain>/<helper>.ts             ← shared pure utilities
```

This separation ensures steps are independently testable and reusable across multiple workflows.

---

## Hooks (Extension Points)

Core-flow workflows expose named hooks at key lifecycle moments:

| Workflow | Hook | Fired after |
|---|---|---|
| `createProductsWorkflow` | `productsCreated` | Products persisted |
| `createOrderWorkflow` | `orderCreated` | Order record created |
| `completeCartWorkflow` | `cartCompleted` | Cart completed, order created |
| `cancelOrderWorkflow` | `orderCanceled` | Order cancelled |
| `createCustomersWorkflow` | `customersCreated` | Customers registered |

---

## Related Packages

- [`@medusajs/framework`](../framework/README.md) — runtime container that executes these workflows
- [`@medusajs/workflows-sdk`](../workflows-sdk/README.md) — composition SDK used to build each workflow
