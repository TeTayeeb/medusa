# customer-identity — Contracts

## Public Interface

```
ports/customer-identity.contract.ts
```

### `CustomerIdentityContract`

```typescript
interface CustomerIdentityContract {
  healthCheck(): Promise<{ module: "customer-identity"; status: "ok" }>
}
```

## API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/store/auth` | Public | Login — create a session/JWT |
| DELETE | `/store/auth` | Customer | Logout — invalidate session |
| POST | `/store/customers` | Public | Register new customer |
| GET | `/store/customers/me` | Customer | Get authenticated customer profile |

## Auth Context Convention

Route handlers MUST receive `AuthenticatedMedusaRequest` (not plain `MedusaRequest`).
Never access `req.auth_context` manually:

```typescript
// ✅ correct
export const GET = async (req: AuthenticatedMedusaRequest, res: MedusaResponse) => {
  const customerId = req.auth_context.actor_id
}

// ❌ incorrect — bypasses framework auth
export const GET = async (req: MedusaRequest, res: MedusaResponse) => {
  const customerId = (req as any).auth_context?.actor_id
}
```

## Contract Stability Rules

All consumers depend on `CustomerIdentityContract`.
Run `yarn test:backend:contracts` after any change.
