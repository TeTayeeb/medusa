# order-fulfillment — Delivery

## Dependencies

| Dependency | Required | Reason |
|---|---|---|
| PostgreSQL | **Yes** | Order, inventory, and fulfillment tables |
| Redis | **Yes** | Workflow step queue; worker polls Redis for pending fulfillment steps |
| `medusa-worker` container | **Yes** | Background fulfillment workflow steps run here |

## Build

```bash
cd apps/backend && yarn build
```

## Database Migrations

Order and fulfillment tables are managed by Medusa core.
No custom migrations unless custom order fields are added.

## Feature Flag

```bash
# .env
FEATURE_ORDER_FULFILLMENT_V2=true
```

## Docker

- Fulfillment HTTP routes (admin) run in `medusa-api`.
- Background workflow steps (inventory reservation, shipping updates) run in `medusa-worker`.

```bash
yarn docker:up
```

To scale workers for high fulfillment volume:

```bash
docker compose up --scale medusa-worker=3 -d
```

## Compensation Functions

All compensation functions MUST be idempotent. After a Medusa version bump,
review compensation steps in any custom fulfillment workflows to ensure
they still match the current order state schema.
