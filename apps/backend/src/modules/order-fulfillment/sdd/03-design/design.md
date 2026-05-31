# order-fulfillment — Design

## Architecture

Port/Adapter pattern:

```
ports/order-fulfillment.contract.ts   ← stable interface
adapters/medusa/                      ← Medusa order/inventory implementation
__tests__/contract.spec.ts            ← type-checked against contract
```

## Port Interface

See `ports/order-fulfillment.contract.ts`.

## Adapter

`OrderFulfillmentMedusaAdapter` uses:
- `@medusajs/order` module for state transitions
- `@medusajs/inventory` module for stock reservation
- `@medusajs/fulfillment` module for shipping provider
- Feature flag: `FEATURE_ORDER_FULFILLMENT_V2`

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/orders` | List orders with fulfillment status |
| GET | `/admin/orders/:id` | Get order with fulfillment details |
| POST | `/admin/orders/:id/fulfillment` | Create fulfillment for order items |
| POST | `/admin/orders/:id/shipment` | Mark fulfillment as shipped |
| POST | `/admin/returns` | Initiate return workflow |

## Upgrade Safety

All mutations go through `core-flows` workflows — never call order module service directly from routes.
