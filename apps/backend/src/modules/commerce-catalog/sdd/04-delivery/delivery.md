# commerce-catalog — Delivery

## Dependencies

| Dependency | Required | Reason |
|---|---|---|
| PostgreSQL | **Yes** | Product, variant, and pricing data |
| Redis | **Yes** | Cache for product queries; workflow state |

## Build

```bash
cd apps/backend && yarn build
```

## Database Migrations

If custom catalog attributes are added to the product model:

```bash
cd apps/backend
yarn medusa db:generate commerceCatalog
yarn medusa db:migrate
```

The one-shot `migrator` Docker container applies all pending migrations on startup.

## Feature Flag

```bash
# .env
FEATURE_COMMERCE_CATALOG_V2=true
```

## Docker

Catalog routes are served by `medusa-api`. The `medusa-worker` handles any
background catalog indexing tasks (Medusa Index Module sync).

```bash
yarn docker:up
```

## Index Module

Medusa's Index Module must be enabled and synced for performant catalog queries.
After a product schema change, re-sync:

```bash
docker compose exec medusa-api yarn medusa exec ./src/scripts/sync-index.ts
```
