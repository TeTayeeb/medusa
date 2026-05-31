# checkout-payment — Validation

## Contract Type Check

```bash
yarn test:backend:contracts
```

Ensures `CheckoutPaymentMedusaAdapter implements CheckoutPaymentContract` at compile time.

## Test File

```
__tests__/contract.spec.ts
```

## Integration Test Scenarios

The following flows should be covered in integration tests:

| Scenario | Expected outcome |
|---|---|
| Happy path: complete cart checkout | Payment captured, order created |
| Duplicate webhook delivery | Idempotent — no double-charge |
| Payment provider returns error | Workflow compensation rolls back cart |
| Concurrent checkout same cart | Second request blocked by Redis lock |
| Cancel payment session | Session removed, cart unlocked |

## Upgrade Safety

```bash
yarn verify:upgrade-safety
```

After a Medusa version bump, check:
- `@medusajs/core-flows` checkout hook signatures
- `@medusajs/payment` provider API changes
