# Software Design Document — `@medusajs/core-flows`

**Version:** 2.15.4  
**Status:** Released  
**Authors:** Medusa Core Team

---

## 1. Purpose & Scope

This document describes the design of `@medusajs/core-flows` — the Medusa built-in workflow library. It covers how workflows and steps are organised, the internal conventions used, compensation strategies, hook injection, and cross-module data orchestration patterns.

---

## 2. Goals & Non-Goals

### Goals
- Provide production-ready, compensable workflows for every commerce domain.
- Allow external consumers to extend workflows via hooks without forking the package.
- Ensure each step is independently testable.
- Keep workflows free of direct database access (all I/O delegated to module services).

### Non-Goals
- Implement module service logic (that belongs in individual module packages).
- Provide an admin UI for workflow monitoring.
- Support workflow-as-a-service (remote execution is handled by `@medusajs/orchestration`).

---

## 3. Structural Conventions

### 3.1 Directory Layout

```
src/
├── cart/
│   ├── steps/          ← createStep() definitions
│   ├── utils/          ← pure helper functions
│   └── workflows/      ← createWorkflow() definitions
├── order/
│   ├── steps/
│   ├── utils/
│   └── workflows/
├── product/
├── payment/
├── payment-collection/
├── customer/
├── fulfillment/
├── auth/
├── promotion/
├── pricing/
├── inventory/
├── reservation/
├── notification/
├── region/
├── file/
├── invite/
├── rbac/
└── index.ts            ← barrel re-export of all workflows and steps
```

### 3.2 Naming Conventions

| Artifact | Pattern | Example |
|---|---|---|
| Workflow file | `<verb>-<noun>.ts` | `create-carts.ts` |
| Step file | `<verb>-<noun>.ts` | `create-carts.ts` (same name, different dir) |
| Workflow ID | `<verb>-<noun>` | `"create-carts"` |
| Step ID | `<verb>-<noun>` | `"create-carts-step"` |
| Export name | camelCase | `createCartsWorkflow` |

---

## 4. Step Design

### 4.1 Step Structure

Every step follows the Invoke + Compensate pattern:

```typescript
export const createCartsStep = createStep(
  "create-carts-step",

  // Invoke: performs the side-effect
  async (input: CreateCartStepInput, { container }) => {
    const cartModule = container.resolve<ICartModuleService>(Modules.CART)
    const carts = await cartModule.createCarts(input.cartsData)
    // Pass cart IDs as compensation data
    return new StepResponse(carts, carts.map((c) => c.id))
  },

  // Compensate: undoes the side-effect on workflow failure
  async (cartIds: string[] | undefined, { container }) => {
    if (!cartIds?.length) return
    const cartModule = container.resolve<ICartModuleService>(Modules.CART)
    await cartModule.deleteCarts(cartIds)
  }
)
```

### 4.2 Compensation Strategy

Core-flows uses **reverse-order compensation**: when a step fails, all successfully-invoked prior steps are compensated in LIFO order. The orchestration engine (`@medusajs/orchestration`) manages this automatically.

Compensation functions:
- MUST be idempotent — they may be called multiple times.
- MUST NOT throw if the resource being reversed no longer exists.
- SHOULD use soft-delete operations when hard-delete would lose audit data.

### 4.3 `useQueryGraphStep`

When a step needs to read data that spans multiple modules (e.g., fetching a cart with its order details and inventory items), it uses `useQueryGraphStep` rather than resolving module services directly:

```typescript
const { data: carts } = useQueryGraphStep({
  entity: "cart",
  fields: ["id", "items.*", "items.variant.inventory_items.*"],
  filters: { id: input.cartId },
})
```

This delegates cross-module joins to the Query engine, keeping steps decoupled from specific module implementations.

---

## 5. Workflow Design

### 5.1 `completeCartWorkflow` — Deep Dive

`completeCartWorkflow` is the most complex workflow in the library. Its step sequence:

```
1. getCartWorkflow (sub-workflow)         — load full cart graph
2. confirmInventoryStep                    — check stock across locations
3. reserveInventoryStep                    — place reservations
4. authorizePaymentSessionWorkflow         — charge customer
5. createOrderWorkflow                     — persist order record
6. createFulfillmentsWorkflow              — allocate shipments
7. refreshCartItemsWorkflow                — clean up cart state
   ─── Hook: cartCompleted ───────────────── fire hook for extensions
```

If `authorizePaymentSessionWorkflow` fails, compensation runs `deleteReservationsStep` (releasing inventory holds) before surfacing the error to the caller.

### 5.2 Parallel Steps

Where order is irrelevant, steps run in parallel via `parallelize()`:

```typescript
const [prices, salesChannels] = parallelize(
  createPriceSetsStep(priceInput),
  attachProductToSalesChannelStep(salesChannelInput)
)
```

Core-flows uses `parallelize` extensively in product creation: variant creation, option creation, and media upload all happen concurrently.

### 5.3 Conditional Steps

Workflows use `when().then()` for conditional paths:

```typescript
const taxLines = when(input, (i) => i.calculateTaxes).then(() =>
  calculateTaxLinesStep(cartData)
)
```

This keeps the workflow graph deterministic and compensable — the `when` block is part of the DAG, not runtime branching.

---

## 6. Hook System

### 6.1 Hook Declaration

Hooks are declared inside the workflow composer function:

```typescript
const orderCreated = createHook("orderCreated", { order })

return new WorkflowResponse(order, { hooks: [orderCreated] })
```

### 6.2 Hook Registration

External code registers handlers at module load time:

```typescript
createOrderWorkflow.hooks.orderCreated(
  async ({ order }, { container }) => {
    // custom side-effect
  }
)
```

Hook handlers:
- Execute **after** the workflow's last step completes successfully.
- Are **not compensated** on workflow failure (they run post-commit).
- Can be registered from plugins, custom modules, or application code.

### 6.3 Available Hooks Catalogue

| Workflow | Hook name | Payload |
|---|---|---|
| `createProductsWorkflow` | `productsCreated` | `{ products }` |
| `updateProductsWorkflow` | `productsUpdated` | `{ products }` |
| `deleteProductsWorkflow` | `productsDeleted` | `{ ids }` |
| `createOrderWorkflow` | `orderCreated` | `{ order }` |
| `cancelOrderWorkflow` | `orderCanceled` | `{ order }` |
| `completeCartWorkflow` | `cartCompleted` | `{ order }` |
| `createCustomersWorkflow` | `customersCreated` | `{ customers }` |
| `createPromotionsWorkflow` | `promotionsCreated` | `{ promotions }` |

---

## 7. Cross-Module Orchestration

Core-flows workflows never import from sibling module packages directly. All module service interaction flows through the DI container:

```typescript
const productModule = container.resolve<IProductModuleService>(Modules.PRODUCT)
```

Cross-module _read_ operations (required for joining data across modules) use `useQueryGraphStep`. Cross-module _write_ relationships are managed via the `RemoteLink` service:

```typescript
const remoteLink = container.resolve(ContainerRegistrationKeys.REMOTE_LINK)
await remoteLink.create([{
  [Modules.PRODUCT]: { product_id: product.id },
  [Modules.SALES_CHANNEL]: { sales_channel_id: salesChannelId },
}])
```

---

## 8. Testing Strategy

### Unit Testing Steps

Each step file can be unit-tested in isolation by mocking the container:

```typescript
import { createCartsStep } from "@medusajs/core-flows"
import { createMockContainer } from "@medusajs/test-utils"

it("creates a cart and returns it", async () => {
  const mockCartModule = { createCarts: jest.fn().mockResolvedValue([{ id: "cart_01" }]) }
  const container = createMockContainer({ [Modules.CART]: mockCartModule })

  const result = await createCartsStep.__invoke({ cartsData: [{ currency_code: "usd" }] }, { container })
  expect(result.output).toEqual([{ id: "cart_01" }])
})
```

### Integration Testing Workflows

Full workflow integration tests run against a real PostgreSQL instance using the `@medusajs/test-utils` bootstrapper (see `integration-tests/http/__tests__/` for examples).

---

## 9. Performance Considerations

- `parallelize()` is used wherever steps have no data dependency, reducing total workflow wall-clock time.
- `useQueryGraphStep` queries use field-projection to retrieve only necessary columns.
- Inventory reservation uses database row-level locking to prevent overselling without distributed locks.
