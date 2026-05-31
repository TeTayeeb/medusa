# @medusajs/payment-stripe

Stripe payment provider for Medusa v2. Implements `IPaymentProvider` via `AbstractPaymentProvider` and registers multiple payment-method sub-services (Cards, iDEAL, Bancontact, BLIK, Giropay, Przelewy24, PromptPay, OXXO) under the `Modules.PAYMENT` namespace.

## Installation

```bash
npm install @medusajs/payment-stripe
```

## Configuration

Register the provider in `medusa-config.ts`:

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/payment",
      options: {
        providers: [
          {
            resolve: "@medusajs/payment-stripe",
            id: "stripe",
            options: {
              apiKey: process.env.STRIPE_API_KEY,          // Stripe secret key (required)
              webhookSecret: process.env.STRIPE_WEBHOOK_SECRET, // Webhook signing secret
              capture: false,                              // false = manual capture (default)
              paymentDescription: "Medusa order",         // Default PI description
              automaticPaymentMethods: true,               // Enable Stripe's automatic PM detection
            },
          },
        ],
      },
    },
  ],
})
```

### Options reference

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `apiKey` | `string` | ✅ | — | Stripe secret key (`sk_live_…` / `sk_test_…`) |
| `webhookSecret` | `string` | — | — | Stripe webhook endpoint signing secret |
| `capture` | `boolean` | — | `false` | `true` → `automatic`, `false` → `manual` capture mode |
| `paymentDescription` | `string` | — | — | Default description added to every PaymentIntent |
| `automaticPaymentMethods` | `boolean` | — | — | Enables `automatic_payment_methods: { enabled: true }` |

## Payment methods supported

| Service class | Provider ID | Payment method |
|---|---|---|
| `StripeProviderService` | `stripe` | Cards / Link |
| `StripeIdealService` | `stripe-ideal` | iDEAL |
| `StripeBancontactService` | `stripe-bancontact` | Bancontact |
| `StripeBlikService` | `stripe-blik` | BLIK |
| `StripeGiropayService` | `stripe-giropay` | Giropay |
| `StripePrzelewy24Service` | `stripe-przelewy24` | Przelewy24 |
| `StripePromptpayService` | `stripe-promptpay` | PromptPay |
| `OxxoProviderService` | `stripe-oxxo` | OXXO |

## Core lifecycle

1. **`initiatePayment`** — Creates a Stripe `PaymentIntent` with amount converted to the currency's smallest unit (e.g. cents). Returns the PI `id` and initial status.
2. **`authorizePayment`** — Calls `getPaymentStatus` to retrieve the current PI state from Stripe; maps it to Medusa's `PaymentSessionStatus`.
3. **`capturePayment`** — Calls `stripe.paymentIntents.capture`. Handles idempotent re-capture gracefully (`PAYMENT_INTENT_UNEXPECTED_STATE` → `succeeded`).
4. **`cancelPayment`** — Calls `stripe.paymentIntents.cancel`. No-ops on already-cancelled intents.
5. **`refundPayment`** — Calls `stripe.refunds.create` with the amount converted to the smallest currency unit.

## Webhook events

Configure the Stripe dashboard to send webhooks to `/hooks/payment/stripe`. The framework routes events by `type`:

| Event | Trigger |
|---|---|
| `payment_intent.succeeded` | Authorises the Medusa payment session |
| `payment_intent.payment_failed` | Marks the session as failed |
| `payment_intent.amount_capturable_updated` | Updates capturable amount |

## Error handling & retries

Network failures (`StripeConnectionError`, `StripeRateLimitError`) are retried up to **3 times** using exponential back-off with jitter. Card errors extract the `payment_intent` from the response so that the session can be reconciled via webhooks. Stripe API errors are treated as indeterminate — the provider does not fail the session and instead awaits a webhook.

## Environment variables

```dotenv
STRIPE_API_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```
