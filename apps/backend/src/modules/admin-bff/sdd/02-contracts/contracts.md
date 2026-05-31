# admin-bff — Contracts

## Public Interface

The stable public interface is defined in:

```
ports/admin-bff.contract.ts
```

### `AdminBffContract`

```typescript
interface AdminBffContract {
  healthCheck(): Promise<{ module: "admin-bff"; status: "ok" }>
}
```

All consumers (API routes, workflows, tests) MUST depend only on `AdminBffContract`.
The concrete class `AdminBffMedusaAdapter` is an implementation detail.

## Contract Stability Rules

1. Methods can be added to the contract — this is backwards compatible.
2. Methods MUST NOT be removed or have their signatures changed without a major version bump.
3. Adapters MUST satisfy the contract at compile time (`tsc --noEmit`).

## Verification

Run `yarn test:backend:contracts` to verify all adapters satisfy their contracts.
