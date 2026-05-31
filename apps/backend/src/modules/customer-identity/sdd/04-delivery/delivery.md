# customer-identity — Delivery

## Dependencies

| Dependency | Required | Reason |
|---|---|---|
| PostgreSQL | **Yes** | Customer and auth records |
| Redis | **Yes** | Session cache (optional but recommended for multi-instance) |

## Build

```bash
cd apps/backend && yarn build
```

## Database Migrations

Customer and auth tables are managed by Medusa core.
No custom migrations unless custom profile fields are added.

## Environment Variables Required

| Variable | Description | Notes |
|---|---|---|
| `JWT_SECRET` | Signs JWT tokens | **Must be identical across all API instances** |
| `COOKIE_SECRET` | Signs session cookies | **Must be identical across all API instances** |
| `AUTH_CORS` | Allowed origins for auth routes | Include storefront origin |
| `STORE_CORS` | Allowed origins for store API | Include storefront origin |

> ⚠ If `JWT_SECRET` or `COOKIE_SECRET` differs between API instances, customers
> will be logged out when load-balanced to a different instance.

## Feature Flag

```bash
# .env
FEATURE_CUSTOMER_IDENTITY_V2=true
```

## Docker

Runs inside `medusa-api`. No dedicated container needed.

```bash
yarn docker:up
```
