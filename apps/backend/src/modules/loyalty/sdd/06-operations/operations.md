# loyalty — Operations

## Running

```bash
yarn docker:up
```

## Feature Flag

```bash
FEATURE_LOYALTY_V2=true
```

## Upgrade Checklist

1. `yarn verify:upgrade-safety`
2. Check `@medusajs/loyalty-plugin` changelog — plugin must be on same release train
3. Re-generate loyalty migrations if schema changed

## Troubleshooting

- **Points not accruing**: verify `order.placed` event subscriber is registered
- **Negative balance**: redemption guard is missing — add validation in workflow step
