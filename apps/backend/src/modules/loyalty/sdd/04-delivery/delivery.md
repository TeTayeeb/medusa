# loyalty — Delivery

## Dependencies

| Dependency | Required | Reason |
|---|---|---|
| PostgreSQL | **Yes** | Loyalty balance and history tables |
| Redis | **Yes** | `order.placed` event consumption via event-bus-redis |
| `@medusajs/workflow-engine-redis` | **Yes** | Background accrual steps run on the worker |

## Build

```bash
cd apps/backend && yarn build
```

## Database Migrations

Loyalty tables are managed by the loyalty plugin.
If the plugin schema changes after a version bump:

```bash
cd apps/backend
yarn medusa db:generate loyalty
yarn medusa db:migrate
```

## Feature Flag

```bash
# .env
FEATURE_LOYALTY_V2=true
```

## Docker

- Loyalty HTTP routes run in `medusa-api`.
- Point accrual (background) runs in `medusa-worker` (subscribes to `order.placed`).

Both containers must be healthy for the full loyalty flow to work.

```bash
yarn docker:up
```

## Version Pinning

The loyalty plugin MUST be on the same release train as `@medusajs/*`.
Include loyalty plugin version in `check-medusa-versions` scope:

```bash
yarn check:medusa-versions
```
