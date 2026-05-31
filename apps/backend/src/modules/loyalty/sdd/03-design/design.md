# loyalty — Design

## Architecture

Port/Adapter pattern:

```
ports/loyalty.contract.ts   ← stable interface
adapters/medusa/             ← Medusa-backed loyalty implementation
__tests__/contract.spec.ts  ← type-checked against contract
```

## Port Interface

See `ports/loyalty.contract.ts`.

## Adapter

`LoyaltyMedusaAdapter` uses:
- `@medusajs/loyalty-plugin` for core loyalty engine (registered as plugin)
- `@medusajs/order` module for purchase event triggers
- Feature flag: `FEATURE_LOYALTY_V2`

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/store/customers/me/loyalty` | Get customer loyalty balance |
| GET | `/admin/loyalty/customers` | Admin view of loyalty balances |
| POST | `/admin/loyalty/adjust` | Manually adjust loyalty points |

## Upgrade Safety

Points are stored as integers (no floating point).
Plugin version must match `@medusajs/*` version — include in `check-medusa-versions` scope.
