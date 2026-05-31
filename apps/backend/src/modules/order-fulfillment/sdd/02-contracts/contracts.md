# order-fulfillment — Contracts

## Public Interface

```
ports/order-fulfillment.contract.ts
```

### `OrderFulfillmentContract`

```typescript
interface OrderFulfillmentContract {
  healthCheck(): Promise<{ module: "order-fulfillment"; status: "ok" }>
}
```

## API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/orders` | Admin | List orders with fulfillment status |
| GET | `/admin/orders/:id` | Admin | Get order with fulfillment details |
| POST | `/admin/orders/:id/fulfillment` | Admin | Create fulfillment for order items |
| POST | `/admin/orders/:id/shipment` | Admin | Mark fulfillment as shipped |
| POST | `/admin/returns` | Admin | Initiate return workflow |

## Workflow Convention

All mutations MUST invoke core-flows workflows:

```typescript
// ✅ correct — goes through workflow (has compensation)
await createFulfillmentWorkflow(req.scope).run({ input: { order_id, items } })

// ❌ incorrect — direct service call, no compensation
const orderService = req.scope.resolve(Modules.ORDER)
await orderService.createFulfillment(...)
```

## Contract Stability Rules

All consumers depend on `OrderFulfillmentContract`.
Run `yarn test:backend:contracts` after any change.
