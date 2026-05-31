# order-fulfillment — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| ORD-01 | The module MUST manage order state transitions: placed → confirmed → fulfillment created → shipped → delivered. |
| ORD-02 | The module MUST reserve inventory when a fulfillment is created. |
| ORD-03 | The module MUST release reserved inventory if fulfillment is cancelled. |
| ORD-04 | The module MUST integrate with a shipping provider to mark orders as shipped. |
| ORD-05 | The module MUST support return initiation and link returns to original orders. |
| ORD-06 | ALL order mutations MUST go through `core-flows` workflows — never direct module service calls from routes. |
| ORD-07 | The module MUST be togglable via `FEATURE_ORDER_FULFILLMENT_V2`. |
| ORD-08 | The module MUST expose a health check returning `{ module: "order-fulfillment", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| ORD-NF-01 | All workflow compensation functions MUST be idempotent. |
| ORD-NF-02 | The `medusa-worker` container MUST be healthy for fulfillment workflow steps to execute. |
| ORD-NF-03 | Worker pods can be scaled independently (`docker compose up --scale medusa-worker=N`). |
| ORD-NF-04 | The module MUST NOT import from Medusa private paths. |

## Out of Scope

- Payment capture (handled by `checkout-payment`)
- Shipping provider implementation (delegated to Medusa fulfillment providers)
- Loyalty point accrual on order events (handled by `loyalty`)
