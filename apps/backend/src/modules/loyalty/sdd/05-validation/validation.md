# loyalty — Validation

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
| Order placed → event fired | Points accrued for customer |
| GET /store/customers/me/loyalty | Returns balance and history |
| Redemption within balance | Checkout discount applied |
| Redemption exceeding balance | 422 — insufficient points |
| Admin manual adjustment (add) | Balance increased |
| Admin manual adjustment (deduct) | Balance decreased, cannot go negative |
| Duplicate order.placed event | Idempotent — points accrued once only |

## Points Integrity Check

After each test run, verify:
- All point values in DB are non-negative integers
- Sum of adjustments matches balance for each customer

## Upgrade Safety

```bash
yarn verify:upgrade-safety
```

After a Medusa version bump, check:
- `@medusajs/loyalty-plugin` changelog for schema changes
- `order.placed` event payload shape changes
- Redemption discount API changes in `@medusajs/discount`
