# customer-identity — Validation

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
| POST /store/customers (new) | Customer created, 201 returned |
| POST /store/auth (valid credentials) | JWT token returned |
| POST /store/auth (bad credentials) | 401 Unauthorized |
| GET /store/customers/me (with token) | Customer profile returned |
| GET /store/customers/me (no token) | 401 Unauthorized |
| DELETE /store/auth | Session invalidated |
| Cross-instance JWT validity | Token issued on instance A accepted by instance B |

## Secret Rotation Procedure

Before rotating `JWT_SECRET`:
1. Deploy new secret to all instances simultaneously (rolling deploy will cause session drops)
2. Notify active users (all active sessions will be invalidated)
3. Verify login works after rotation

## Upgrade Safety

```bash
yarn verify:upgrade-safety
```

After a Medusa version bump, check:
- `@medusajs/auth` provider API changes
- `AuthenticatedMedusaRequest` type changes
- CORS configuration requirements
