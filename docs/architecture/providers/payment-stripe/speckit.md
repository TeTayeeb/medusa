# SpecKit — @medusajs/payment-stripe

Test specifications and acceptance criteria for the Stripe payment provider.

---

## 1. Unit specs — `StripeBase`

### 1.1 `initiatePayment`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Happy path — card payment | `{ amount: 100, currency_code: "usd", data: {} }` | Returns `{ id: "pi_...", status: "pending" }` |
| U2 | Smallest-unit conversion | `amount: 9.99, currency_code: "usd"` | Stripe called with `amount: 999` |
| U3 | JPY (zero-decimal currency) | `amount: 500, currency_code: "jpy"` | Stripe called with `amount: 500` (no multiplication) |
| U4 | Capture mode — automatic | `options.capture: true` | PI created with `capture_method: "automatic"` |
| U5 | Capture mode — manual (default) | `options.capture: false` | PI created with `capture_method: "manual"` |
| U6 | Idempotency key forwarded | `context.idempotency_key: "ikey-123"` | Stripe SDK called with `{ idempotencyKey: "ikey-123" }` |

### 1.2 `capturePayment`

| # | Scenario | Expected outcome |
|---|---|---|
| U7 | Normal capture | Returns `{ data: <stripeIntent> }` with `status: "succeeded"` |
| U8 | Already captured (`PAYMENT_INTENT_UNEXPECTED_STATE` + `succeeded`) | Returns the existing `payment_intent` without throwing |
| U9 | Missing PI id in data | Does not call Stripe; returns `{ data }` unchanged |

### 1.3 `refundPayment`

| # | Scenario | Expected outcome |
|---|---|---|
| U10 | Partial refund | `stripe.refunds.create` called with correct smallest-unit amount |
| U11 | Full refund | Refund amount equals full PI amount in smallest unit |
| U12 | Already refunded (`charge_already_refunded`) | `MedusaError` thrown |

### 1.4 `cancelPayment`

| # | Scenario | Expected outcome |
|---|---|---|
| U13 | Normal cancel | Returns `{ data: <cancelledIntent> }` |
| U14 | Already cancelled | Stripe error caught; returns existing PI data (no throw) |

### 1.5 `executeWithRetry`

| # | Scenario | Expected outcome |
|---|---|---|
| U15 | `StripeConnectionError` on first call | Retried; succeeds on second attempt |
| U16 | `StripeRateLimitError` — 3 retries exhausted | Throws `MedusaError` after 3rd failure |
| U17 | `StripeCardError` with `payment_intent` | Returns PI data without retry |
| U18 | `StripeAPIError` | Returns `{ indeterminate_due_to: "stripe_api_error" }` |

---

## 2. Unit specs — status mapping

| # | Stripe PI status | Expected `PaymentSessionStatus` |
|---|---|---|
| U19 | `requires_payment_method` | `pending` |
| U20 | `requires_action` | `requires_more` |
| U21 | `requires_capture` | `authorized` |
| U22 | `succeeded` | `captured` |
| U23 | `canceled` | `canceled` |
| U24 | `processing` | `pending` |

---

## 3. Unit specs — sub-providers

| # | Service | `paymentIntentOptions` |
|---|---|---|
| U25 | `StripeIdealService` | `payment_method_types: ["ideal"]` |
| U26 | `StripeBancontactService` | `payment_method_types: ["bancontact"]` |
| U27 | `OxxoProviderService` | `payment_method_types: ["oxxo"]` |

---

## 4. Integration specs

| # | Scenario | Setup | Expected outcome |
|---|---|---|---|
| I1 | Full happy-path purchase | Live Stripe test keys, cart with items | Payment session created → authorised → captured |
| I2 | Webhook `payment_intent.succeeded` | Post valid Stripe webhook | Payment session status updated to `captured` |
| I3 | Webhook `payment_intent.payment_failed` | Post failure webhook | Payment session status updated to `error` |
| I4 | 3DS required | Card requiring authentication | Status becomes `requires_more` |

---

## 5. Configuration validation

| # | Scenario | Expected outcome |
|---|---|---|
| C1 | `apiKey` omitted | Throws `Error("Required option 'apiKey' is missing")` at boot |
| C2 | Valid config | Provider boots without errors |

---

## 6. Acceptance criteria

- All smallest-unit conversions are correct for all ISO 4217 currencies.
- Retries never exceed 3 attempts; final failure surfaces a `MedusaError`.
- Idempotency keys are always forwarded to Stripe on mutating calls.
- Webhook events update payment session status within 500 ms of delivery.
- Sub-providers each register with their declared `identifier`.
