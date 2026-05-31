# admin-bff — Validation

## Contract Type Check

The primary validation mechanism is the TypeScript contract test:

```bash
yarn test:backend:contracts
# runs: cd apps/backend && tsc --noEmit --project tsconfig.json
```

This ensures `AdminBffMedusaAdapter` satisfies `AdminBffContract` at compile time.

## Test File

```
__tests__/contract.spec.ts
```

The spec imports `AdminBffMedusaAdapter` and asserts it is assignable to `AdminBffContract`.
No runtime tests are needed — the type check is sufficient for the contract.

## Upgrade Safety Check

```bash
yarn verify:upgrade-safety
```

Runs the full pipeline: version consistency → boundary lint → contract typecheck → build.

## Manual Verification

After deploying:

```bash
curl http://localhost:9000/health
# expected: { "status": "ok" }
```
