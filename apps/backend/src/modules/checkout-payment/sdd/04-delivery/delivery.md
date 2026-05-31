# checkout-payment — Delivery

## Dependencies

| Dependency | Required | Reason |
|---|---|---|
| Redis | **Yes** | Distributed locking for concurrent checkout prevention; workflow state |
| PostgreSQL | **Yes** | Cart and payment session persistence |

## Build

```bash
cd apps/backend && yarn build
```

## Database Migrations

The module relies on Medusa's built-in cart and payment tables.
No custom migrations are needed unless custom checkout fields are added.

If custom fields are added:
```bash
cd apps/backend && yarn medusa db:generate checkoutPayment
yarn medusa db:migrate
```

## Feature Flag

```bash
# .env
FEATURE_CHECKOUT_PAYMENT_V2=true
```

## Docker

Runs in `medusa-api` (HTTP) and `medusa-worker` (background compensation steps):

```bash
yarn docker:up
```

Both containers must be healthy for full checkout flows to work.
The worker handles compensation/rollback steps on payment failure.

## Environment Variables Required

| Variable | Description |
|---|---|
| `REDIS_URL` | Redis connection (required for distributed lock) |
| `DATABASE_URL` | PostgreSQL connection |
