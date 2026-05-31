# checkout-payment — Design

## Architecture

Port/Adapter pattern:

```
ports/checkout-payment.contract.ts   ← stable interface
adapters/medusa/                     ← Medusa-backed implementation
__tests__/contract.spec.ts           ← type-checked against contract
```

## Port Interface

See `ports/checkout-payment.contract.ts`.

## Adapter

`CheckoutPaymentMedusaAdapter` uses:
- `@medusajs/medusa` cart and payment workflow hooks
- Feature flag: `FEATURE_CHECKOUT_PAYMENT_V2`

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/store/carts/:id/complete` | Complete checkout and capture payment |
| POST | `/store/payment-sessions` | Initialize a payment session |
| DELETE | `/store/payment-sessions/:id` | Cancel a payment session |

## Upgrade Safety

Adapter only uses `@medusajs/framework` and `@medusajs/core-flows` public exports.
