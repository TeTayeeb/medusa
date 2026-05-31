# loyalty — Contracts

## Public Interface

```
ports/loyalty.contract.ts
```

### `LoyaltyContract`

```typescript
interface LoyaltyContract {
  healthCheck(): Promise<{ module: "loyalty"; status: "ok" }>
}
```

## API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/store/customers/me/loyalty` | Customer | Get loyalty balance and history |
| GET | `/admin/loyalty/customers` | Admin | Admin view of all customer loyalty balances |
| POST | `/admin/loyalty/adjust` | Admin | Manually adjust points for a customer |

## Points Rules

- Points are stored as non-negative integers.
- Redemption MUST be validated in a workflow step before checkout completion:
  ```
  balance >= redemption_amount
  ```
- Accrual is triggered by `order.placed` event (handled by the worker).

## Contract Stability Rules

All consumers depend on `LoyaltyContract`.
Run `yarn test:backend:contracts` after any change.
