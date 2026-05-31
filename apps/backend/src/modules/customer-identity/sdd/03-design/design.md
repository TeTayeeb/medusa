# customer-identity — Design

## Architecture

Port/Adapter pattern:

```
ports/customer-identity.contract.ts   ← stable interface
adapters/medusa/                      ← Medusa auth/customer implementation
__tests__/contract.spec.ts            ← type-checked against contract
```

## Port Interface

See `ports/customer-identity.contract.ts`.

## Adapter

`CustomerIdentityMedusaAdapter` uses:
- `@medusajs/auth` module for authentication
- `@medusajs/customer` module for profiles
- Feature flag: `FEATURE_CUSTOMER_IDENTITY_V2`

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/store/auth` | Login / create session |
| DELETE | `/store/auth` | Logout |
| GET | `/store/customers/me` | Get authenticated customer profile |
| POST | `/store/customers` | Register new customer |

## Upgrade Safety

Route handlers use `AuthenticatedMedusaRequest` — never check `req.auth_context` manually.
