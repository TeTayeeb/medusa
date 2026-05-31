# checkout-payment — Contracts

## Public Interface

```
ports/checkout-payment.contract.ts
```

### `CheckoutPaymentContract`

```typescript
interface CheckoutPaymentContract {
  healthCheck(): Promise<{ module: "checkout-payment"; status: "ok" }>
}
```

## Planned Extensions

When new operations are added to the contract, add them to this interface first,
then implement in `CheckoutPaymentMedusaAdapter`. Do not use the adapter directly
from consumers.

## API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/store/carts/:id/complete` | Customer | Complete checkout and capture payment |
| POST | `/store/payment-sessions` | Customer | Initialize a payment session |
| DELETE | `/store/payment-sessions/:id` | Customer | Cancel a payment session |

## Contract Stability Rules

1. Adding methods is backwards compatible.
2. Removing or renaming methods requires a version bump.
3. Run `yarn test:backend:contracts` to verify after changes.
