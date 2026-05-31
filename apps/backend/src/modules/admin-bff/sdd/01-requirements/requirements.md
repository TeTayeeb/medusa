# admin-bff — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| ABF-01 | The module MUST aggregate data from multiple Medusa API endpoints into a single admin-optimized response. |
| ABF-02 | The module MUST transform Medusa data models into admin UI DTOs without exposing raw module internals. |
| ABF-03 | The module MUST expose only routes mounted under `/admin/` (no store-facing routes). |
| ABF-04 | The module MUST be independently togglable via `FEATURE_ADMIN_BFF_V2` without requiring a re-deploy. |
| ABF-05 | The module MUST expose a `/health` endpoint returning `{ module: "admin-bff", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| ABF-NF-01 | All data aggregation MUST complete within 500 ms under normal load. |
| ABF-NF-02 | The module MUST NOT import from `@medusajs/*/dist/*` or `@medusajs/*/src/*` — public API surface only. |
| ABF-NF-03 | The module contract MUST pass `tsc --noEmit` on every CI run. |

## Out of Scope

- Authentication and authorization (handled by `customer-identity`)
- Raw product/order data mutations (handled by `commerce-catalog` / `order-fulfillment`)
