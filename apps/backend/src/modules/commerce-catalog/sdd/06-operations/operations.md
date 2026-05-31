# commerce-catalog — Operations

## Running

```bash
yarn docker:up
```

## Feature Flag

```bash
FEATURE_COMMERCE_CATALOG_V2=true
```

## Upgrade Checklist

1. `yarn verify:upgrade-safety`
2. Check `@medusajs/product` changelog for data model changes
3. Re-generate migrations if product schema changed: `medusa db:generate commerceCatalog`

## Troubleshooting

- **Missing products in query**: verify Index Module is enabled and synced
- **Price not returned**: ensure `fields` includes `variants.prices` in query config
