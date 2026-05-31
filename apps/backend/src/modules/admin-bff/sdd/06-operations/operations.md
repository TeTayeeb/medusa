# admin-bff — Operations

## Running Locally

```bash
yarn docker:up           # start full stack
# or for local dev without Docker:
cd apps/backend && yarn dev
```

## Feature Flag

```bash
# .env
FEATURE_ADMIN_BFF_V2=true
```

## Upgrade Checklist

1. Run `yarn verify:upgrade-safety`
2. Ensure contract tests pass: `yarn test:backend:contracts`
3. Update adapter if Medusa API changed — contract interface stays the same

## API Routes

All admin-bff routes are mounted under `/admin/` via `apps/backend/src/api/`.

See `docs/architecture/ARCHITECTURE_ROADMAP.md` for the full system context.
