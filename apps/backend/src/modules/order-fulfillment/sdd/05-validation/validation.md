# order-fulfillment — Validation

## Contract Type Check

```bash
yarn test:backend:contracts
```

## Test File

```
__tests__/contract.spec.ts
```

## Integration Test Scenarios

| Scenario | Expected outcome |
|---|---|
| POST /admin/orders/:id/fulfillment | Fulfillment created, inventory reserved |
| POST /admin/orders/:id/shipment | Order status → shipped, tracking info stored |
| Fulfillment cancelled | Inventory reservation released via compensation |
| POST /admin/returns | Return workflow initiated, return object created |
| Worker unhealthy (Redis unreachable) | Pending steps remain queued, no data loss |
| Duplicate fulfillment step | Idempotent — compensation handles re-run safely |

## Compensation Idempotency Check

Before release, verify that each compensation function handles:
1. Being called on already-compensated state (no-op)
2. Partial failure mid-compensation (retry safe)

## Upgrade Safety

```bash
yarn verify:upgrade-safety
```

After a Medusa version bump, check:
- `@medusajs/order` state machine changes
- `@medusajs/inventory` reservation API changes
- `@medusajs/fulfillment` shipping provider interface changes
- Compensation function signatures in any custom core-flows steps
