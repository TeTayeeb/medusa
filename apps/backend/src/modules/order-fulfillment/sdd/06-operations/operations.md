# order-fulfillment — Operations

## Running

```bash
yarn docker:up
```

## Feature Flag

```bash
FEATURE_ORDER_FULFILLMENT_V2=true
```

## Upgrade Checklist

1. `yarn verify:upgrade-safety`
2. Check `@medusajs/order` and `@medusajs/fulfillment` changelogs
3. Review compensation functions on fulfillment steps after upgrade

## Scaling

The `medusa-worker` handles all fulfillment workflow steps asynchronously.
Scale workers independently:
```bash
docker compose up --scale medusa-worker=3
```

## Troubleshooting

- **Inventory not reserved**: check `medusa-worker` logs for failed step
- **Fulfillment stuck**: verify Redis is reachable from `medusa-worker`
- **Return workflow error**: ensure compensation functions match current order state schema
