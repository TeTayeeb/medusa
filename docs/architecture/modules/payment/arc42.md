# arc42 Architecture Documentation — Payment Module

> Medusa v2.15.4 | Focused sections: 1, 3, 5, 6, 8, 9

---

## Section 1 — Introduction and Goals

### 1.1 Purpose

The Payment Module is the **provider-agnostic financial processing layer** of Medusa. Its core purpose is to abstract all payment provider differences behind a uniform interface so that merchant-facing code (workflows, API routes, admin UI) does not need to know whether it is talking to Stripe, PayPal, Klarna, or any other PSP.

**Stakeholders:**
- **Merchants** — capture payments, issue refunds, view payment status
- **Customers** — initiate payment sessions, complete checkout
- **Payment providers** — implement `IPaymentProvider`; receive dispatched calls
- **Order module** — receives `order.paid` signal to advance order status
- **Finance teams** — consume `Payment`, `Capture`, `Refund` records for reconciliation

### 1.2 Quality Goals

| Priority | Goal | Scenario |
|---|---|---|
| 1 | **Reliability** | A failed provider call must not leave the system in a partially committed state |
| 2 | **Provider Isolation** | Adding or removing a provider must require zero changes to the Payment module core |
| 3 | **Financial Accuracy** | All amounts must use BigNumber arithmetic; no floating-point sums |
| 4 | **Security** | Provider credentials must never be logged; webhook signatures must be verified |

---

## Section 3 — Building Block View

### Level 1 — Context

```
┌────────────────────────────────────────────────────────┐
│                    Medusa Application                   │
│                                                         │
│  Store API ──▶ addPaymentSessionsWorkflow               │
│  Admin API ──▶ capturePaymentWorkflow                   │
│  Webhook   ──▶ processPaymentWorkflow                   │
│                      │                                  │
│            PaymentModuleService                        │
│                      │                                  │
│            PaymentProviderService                      │
│              ┌───────┴────────────────┐                │
│              │  pp_stripe_stripe      │  External PSPs  │
│              │  pp_paypal_paypal      │◀─────────────── │
│              │  pp_system_default     │                 │
│              └────────────────────────┘                │
│                      │                                  │
│              PostgreSQL (MikroORM)                      │
│  Event Bus ◀── payment.captured / payment.refunded ...  │
└────────────────────────────────────────────────────────┘
```

### Level 2 — Key Components

| Component | Responsibility |
|---|---|
| `PaymentModuleService` | Orchestrates payment lifecycle; manages all entity state |
| `PaymentProviderService` | Provider registry; dispatches to the correct `IPaymentProvider` |
| Provider implementations | External packages (e.g. `@medusajs/payment-stripe`) |
| `loaders/providers.ts` | Dynamically loads and registers providers at startup |
| `IPaymentProvider` interface | Contract all providers must implement |

---

## Section 5 — Runtime View: Authorize Payment Session

```
Customer → POST /store/carts/:id/complete
    │
    ├─ completeCartWorkflow
    │       │
    │       ├─ [step] authorizePaymentSessionStep
    │       │      └─ PaymentModuleService.authorizePaymentSession(sessionId, context)
    │       │              │
    │       │              ├─ retrieve PaymentSession (validate status = pending)
    │       │              ├─ PaymentProviderService.authorizePayment(session)
    │       │              │      └─ provider.authorizePayment({ data, context })
    │       │              │              └─ calls external PSP API
    │       │              │
    │       │              ├─ if status = authorized:
    │       │              │      ├─ INSERT Payment record
    │       │              │      ├─ UPDATE PaymentSession.status = authorized
    │       │              │      └─ UPDATE PaymentCollection.authorized_amount
    │       │              │
    │       │              └─ if status = requires_more:
    │       │                     └─ UPDATE PaymentSession.status = requires_more
    │       │                     └─ return (checkout paused for 3DS etc.)
    │       │
    │       └─ [step] createOrderFromCartStep (only if authorized)
```

---

## Section 6 — Runtime View: Capture Payment

```
Merchant → POST /admin/payments/:id/capture { amount? }
    │
    ├─ capturePaymentWorkflow.run({ payment_id, amount })
    │       │
    │       ├─ [step] capturePaymentStep
    │       │      └─ PaymentModuleService.capturePayment({ payment_id, amount })
    │       │              │
    │       │              ├─ retrieve Payment (validate not canceled)
    │       │              ├─ validate capture amount ≤ remaining capturable
    │       │              ├─ PaymentProviderService.capturePayment(payment)
    │       │              │      └─ provider.capturePayment({ data, amount })
    │       │              │              └─ external PSP charge
    │       │              │
    │       │              ├─ INSERT Capture record
    │       │              ├─ UPDATE Payment.captured_at (if fully captured)
    │       │              ├─ recalculate PaymentCollection.captured_amount
    │       │              └─ derive PaymentCollection.status
    │       │
    │       └─ emit payment.captured
```

---

## Section 8 — Crosscutting Concepts

### Provider Isolation via IPaymentProvider

Every provider interaction is dispatched through `PaymentProviderService.{action}()` which:
1. Looks up the registered `IPaymentProvider` by `provider_id`
2. Normalises the input (currency precision, amount formatting)
3. Calls the provider method
4. Wraps provider errors in `MedusaError`

This ensures the `PaymentModuleService` never imports provider-specific code.

### Amount Precision

`defaultCurrencies` maps every ISO 4217 code to its `decimal_digits` value. Amounts are multiplied by `10^decimal_digits` for integer representation when communicating with providers. `getEpsilonFromDecimalPrecision()` provides floating-point safe comparison for amount validation.

### Webhook Processing

`POST /hooks/payment/:provider_id` routes to `processPaymentWorkflow`. The workflow calls `provider.getWebhookActionAndData()` to normalise the webhook payload into a `WebhookActionResult` (`captured`, `refunded`, `authorized`, `failed`). The workflow then dispatches to `capturePayment` / `refundPayment` etc. as needed.

### AccountHolder (Saved Payment Methods)

`AccountHolder` links a `customer_id` to a provider-specific `account_id` (e.g. Stripe `cus_xxx`). Providers implementing the optional account holder interface can list saved payment methods for returning customers.

---

## Section 9 — Architecture Decisions

### ADR-PAY-001: IPaymentProvider as Strict Interface

**Decision**: All payment providers must implement `IPaymentProvider` as a **typed interface**, not a base class.  
**Rationale**: Allows providers to be developed as independent packages with their own dependencies. Avoids tight coupling to Medusa internals.  
**Consequence**: The interface is a **stability contract** — breaking changes require major version bumps with migration guides.

### ADR-PAY-002: PaymentCollection as Multi-Session Container

**Decision**: One `PaymentCollection` can hold multiple `PaymentSession` records (one per provider).  
**Rationale**: Merchants may offer multiple payment providers (Stripe + PayPal); customers pick one per checkout.  
**Consequence**: Amount tracking must aggregate across all sessions/payments in the collection.

### ADR-PAY-003: Capture as Separate Entity

**Decision**: Each capture event creates a `Capture` entity rather than updating `Payment.amount`.  
**Rationale**: Supports **partial captures** (e.g. ship one item, capture its portion); provides full audit log of financial events.  
**Consequence**: `PaymentCollection.captured_amount` must be recomputed as a sum on every capture.

### ADR-PAY-004: Webhook Normalisation in Provider

**Decision**: Webhook payload parsing and action mapping lives in the **provider** (`getWebhookActionAndData()`), not in the module.  
**Rationale**: Each PSP has a completely different webhook format; provider knows its own format best.  
**Consequence**: The core module can process any webhook without knowing provider-specific formats.
