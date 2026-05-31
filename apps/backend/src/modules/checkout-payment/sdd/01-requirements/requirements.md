# checkout-payment — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| CPY-01 | The module MUST orchestrate the full cart → checkout → payment capture flow. |
| CPY-02 | The module MUST validate business rules (e.g. cart non-empty, payment method present) before capture. |
| CPY-03 | The module MUST handle payment provider webhooks and update payment status accordingly. |
| CPY-04 | The module MUST roll back cart and inventory reservations via workflow compensation if payment capture fails. |
| CPY-05 | The module MUST support initializing and cancelling payment sessions. |
| CPY-06 | The module MUST be independently togglable via `FEATURE_CHECKOUT_PAYMENT_V2`. |
| CPY-07 | The module MUST expose a health check returning `{ module: "checkout-payment", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| CPY-NF-01 | Payment capture MUST be idempotent — duplicate webhook deliveries MUST NOT result in double-charges. |
| CPY-NF-02 | Distributed locking via Redis MUST prevent concurrent checkout completion on the same cart. |
| CPY-NF-03 | Workflow compensation steps MUST be idempotent to handle retries safely. |
| CPY-NF-04 | The module MUST NOT import from Medusa private paths (`*/dist/*`, `*/src/*`). |

## Out of Scope

- Payment provider implementation (delegated to Medusa payment providers)
- Cart item management (handled by the cart module)
- Refunds and returns (handled by `order-fulfillment`)
