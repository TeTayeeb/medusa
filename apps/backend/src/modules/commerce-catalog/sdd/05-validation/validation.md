# commerce-catalog — Validation

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
| GET /store/products | Returns paginated product list with prices |
| GET /store/products/:id | Returns product with variants and prices |
| GET /admin/products (filtered) | Filtered list with extended metadata |
| POST /admin/products | Product created with catalog attributes |
| Missing `variants.prices` field | Caught by price inclusion test |

## Query Convention Check

Lint rule (`lint:backend-boundaries`) ensures no direct module service imports
from API route files. All reads must go through `query.graph()`.

## Upgrade Safety

```bash
yarn verify:upgrade-safety
```

After a Medusa version bump, check:
- `@medusajs/product` data model changes
- `@medusajs/pricing` price list API changes
- Query `fields` paths (check for renamed relations)
