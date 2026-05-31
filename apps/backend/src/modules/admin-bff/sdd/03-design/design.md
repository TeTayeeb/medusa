# admin-bff — Design

## Architecture

The module follows the **Port/Adapter pattern**:

```
ports/admin-bff.contract.ts       ← stable interface, no Medusa dependency
adapters/medusa/                  ← Medusa implementation, swappable
__tests__/contract.spec.ts        ← type-checked against contract
```

## Port Interface

See `ports/admin-bff.contract.ts`.

All consumers depend only on `AdminBffContract`, never on the adapter directly.

## Adapter

`AdminBffMedusaAdapter` implements `AdminBffContract` using:
- `@medusajs/framework` public APIs only (no deep imports)
- Medusa DI container for module resolution
- Feature flag: `FEATURE_ADMIN_BFF_V2`

## Feature Flag

Set `FEATURE_ADMIN_BFF_V2=true` in `.env` to enable v2 logic.

## Upgrade Safety

- No `@medusajs/*/dist/*` or `@medusajs/*/src/*` imports
- Contract shape tested via `tsc --noEmit` on every `verify:upgrade-safety` run
