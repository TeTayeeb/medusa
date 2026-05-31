# loyalty — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| LOY-01 | The module MUST accrue loyalty points when an `order.placed` event is received. |
| LOY-02 | The module MUST expose a customer-facing balance and history endpoint. |
| LOY-03 | The module MUST allow point redemption during checkout (applied as a discount). |
| LOY-04 | The module MUST validate that redemption does not result in a negative balance. |
| LOY-05 | The module MUST support admin-initiated manual point adjustments. |
| LOY-06 | The module MUST expire unused points based on a configurable policy. |
| LOY-07 | The module MUST be togglable via `FEATURE_LOYALTY_V2`. |
| LOY-08 | The module MUST expose a health check returning `{ module: "loyalty", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| LOY-NF-01 | Points MUST be stored as integers (no floating point arithmetic). |
| LOY-NF-02 | Redemption validation MUST occur in a workflow step with compensation to handle rollback. |
| LOY-NF-03 | The loyalty plugin version MUST match the `@medusajs/*` monorepo version (same release train). |
| LOY-NF-04 | The module MUST NOT import from Medusa private paths. |

## Out of Scope

- Payment provider integration (delegated to `checkout-payment`)
- Order creation (delegated to Medusa's order module)
- Gift cards (separate feature)
