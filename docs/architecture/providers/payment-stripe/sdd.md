# Software Design Document — @medusajs/payment-stripe

## 1. Purpose

Provide a production-grade Stripe integration for Medusa's Payment Module. The provider translates Medusa's generic payment lifecycle (initiate → authorise → capture → refund / cancel) into Stripe PaymentIntent operations, while supporting multiple localised payment methods as independent sub-providers.

## 2. Architecture overview

```
Modules.PAYMENT
  └── ModuleProvider (payment-stripe)
        ├── StripeBase (AbstractPaymentProvider)
        │     ├── initiatePayment
        │     ├── authorizePayment
        │     ├── capturePayment
        │     ├── cancelPayment / deletePayment
        │     ├── refundPayment
        │     ├── getPaymentStatus
        │     └── executeWithRetry  ← exponential back-off
        ├── StripeProviderService   (identifier: "stripe")
        ├── StripeIdealService      (identifier: "stripe-ideal")
        ├── StripeBancontactService (identifier: "stripe-bancontact")
        ├── StripeBlikService       (identifier: "stripe-blik")
        ├── StripeGiropayService    (identifier: "stripe-giropay")
        ├── StripePrzelewy24Service (identifier: "stripe-przelewy24")
        ├── StripePromptpayService  (identifier: "stripe-promptpay")
        └── OxxoProviderService     (identifier: "stripe-oxxo")
```

Each sub-service extends `StripeBase` and overrides only `paymentIntentOptions` (a getter returning method-specific PI parameters such as `payment_method_types`). The common logic lives entirely in `StripeBase`.

## 3. Key design decisions

### 3.1 Smallest-unit conversion
Stripe requires amounts in the currency's indivisible unit (cents for USD, yen as-is for JPY). `getSmallestUnit(amount, currencyCode)` applies the correct multiplier per ISO 4217 exponent. This is invoked in `initiatePayment` and `refundPayment`.

### 3.2 Retry strategy
`executeWithRetry` wraps every Stripe API call. It distinguishes three error categories:
- **Retryable** (`StripeConnectionError`, `StripeRateLimitError`) — retry up to 3 times, delay = `baseDelay × 2^(attempt-1) × rand(0.5–1.0)`.
- **Card errors with PI data** — return the PI object (non-retryable). Webhook will reconcile.
- **Indeterminate API errors** — return a sentinel object; do not throw, do not retry.
- **Fatal errors** — rethrow via `buildError`.

### 3.3 Idempotency
Every mutating Stripe call passes `context.idempotency_key` as the Stripe idempotency key, ensuring at-most-once semantics for network retries above the transport layer.

### 3.4 Capture modes
`capture: true` in provider options maps to `capture_method: "automatic"`, creating and capturing in one step. `capture: false` (default) uses `capture_method: "manual"`, requiring an explicit `capturePayment` call. Individual PI creation can override this via `data.capture_method`.

### 3.5 Status mapping
`getStatus` maps Stripe PI statuses to `PaymentSessionStatus`:
| Stripe status | Medusa status |
|---|---|
| `requires_payment_method` | `pending` |
| `requires_action` | `requires_more` |
| `requires_capture` | `authorized` |
| `succeeded` | `captured` |
| `canceled` | `canceled` |
| `processing` | `pending` |

## 4. Data flow

```
Client → POST /store/carts/:id/payment-sessions
  → PaymentModule.createPaymentSession
  → StripeBase.initiatePayment
  → stripe.paymentIntents.create (amount in smallest unit)
  → { id, status } stored in payment_session.data
```

```
Stripe Webhook → POST /hooks/payment/stripe
  → PaymentModule.processEvent
  → StripeBase.getWebhookActionAndData
  → PaymentModule.authorizePaymentSession | setPaymentSessionStatus
```

## 5. Error propagation

All thrown errors use `this.buildError(message, stripeError)` which wraps the original Stripe error into a `MedusaError` preserving the original cause. The Payment Module catches these and exposes them to calling workflows, which can then compensate (e.g. cancel the payment session).

## 6. Multi-PM sub-providers

Each sub-service is a thin wrapper that only specifies the `payment_method_types` array and optionally additional PI parameters. The `ModuleProvider` factory registers all eight services so the Medusa admin can expose each as a distinct payment option per region.

## 7. Webhook verification

Webhook events should be verified using `stripe.webhooks.constructEvent(body, sig, webhookSecret)` before processing. The provider trusts the framework to handle verification at the HTTP layer before dispatching `getWebhookActionAndData`.

## 8. Sequence: PaymentIntent creation

See `diagrams/flow.mmd` for the full initiatePayment → capture flow.

## 9. Dependencies

| Package | Purpose |
|---|---|
| `stripe` | Official Stripe Node.js SDK |
| `@medusajs/framework` | `AbstractPaymentProvider`, `MedusaError`, `ModuleProvider` |
| `timers/promises` | `setTimeout` for retry back-off |
