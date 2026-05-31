# admin-bff — Delivery

## Build

No additional build step beyond the standard `medusa build`.

```bash
cd apps/backend && yarn build
```

## Database Migrations

The `admin-bff` module owns no database tables. No migrations required.

## Feature Flag

```bash
# .env
FEATURE_ADMIN_BFF_V2=true   # enable v2 BFF logic
```

The flag defaults to `false`. Set it to `true` to activate the v2 adapter.

## Docker

The module runs inside the `medusa-api` container (no dedicated container needed).

```bash
yarn docker:up   # starts full stack including admin-bff routes
```

## Packaging

The module ships as source TypeScript inside `apps/backend`. There is no separate npm package.
It is part of the monorepo and built together with the backend.
