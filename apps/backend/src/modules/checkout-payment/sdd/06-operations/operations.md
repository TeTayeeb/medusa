# checkout-payment — Operations

## Running

```bash
yarn docker:up
```

## Feature Flag

```bash
FEATURE_CHECKOUT_PAYMENT_V2=true
```

## Upgrade Checklist

1. `yarn verify:upgrade-safety`
2. Review `@medusajs/core-flows` checkout hooks for breaking changes
3. Update adapter if payment session API changed

## Troubleshooting

- **Payment session not found**: ensure `REDIS_URL` is set and Redis is healthy
- **Cart locked**: distributed lock held by another request — retry after 5s
