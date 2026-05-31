# Payment Module — `@medusajs/payment`

> Medusa v2.15.4 · Module Reference

## Purpose & Domain Responsibility

The Payment Module is the **financial gateway abstraction layer** for Medusa. It manages the complete payment lifecycle — from session initiation through authorization, capture, and refund — while remaining agnostic to any specific payment provider. It defines a standard `IPaymentProvider` interface that all providers implement (Stripe, PayPal, Klarna, etc.).

The module orchestrates `PaymentCollection` (the container for all payment activity against an order/cart), `PaymentSession` (a single provider-specific authorization attempt), `Payment` (a committed payment record), and `Refund` (a partial or full reversal).

---

## Key Entities

| Entity | Prefix | Description |
|---|---|---|
| `PaymentCollection` | `pay_col_` | Container linking all payment activity to a resource (cart/order). Tracks total `amount`, `authorized_amount`, `captured_amount`, `refunded_amount`, and overall `status`. |
| `PaymentSession` | `payses_` | A single payment authorization attempt with a specific provider. Holds provider-specific `data` and session `status` (pending → authorized / error / canceled). |
| `Payment` | `pay_` | A committed, authorized payment record. Created when a session is authorized. Links back to both `PaymentCollection` and `PaymentSession`. |
| `Capture` | `cap_` | A capture event against a payment. A single payment can have multiple partial captures. |
| `Refund` | `ref_` | A refund event against a payment. Partial or full. Links to `RefundReason`. |
| `RefundReason` | `rr_` | A catalogue of standard refund reasons (duplicate, fraudulent, customer request, etc.). |
| `PaymentProvider` | — | Registration record of available payment providers with `id` (e.g. `pp_stripe_stripe`). |
| `AccountHolder` | — | An account holder record linking a customer to provider-specific customer data (e.g. Stripe Customer ID). Enables saved payment methods. |

---

## Key Service Methods

```ts
// PaymentCollection
createPaymentCollections(data[], context): Promise<PaymentCollectionDTO[]>
updatePaymentCollections(id, data, context): Promise<PaymentCollectionDTO>
completePaymentCollections(id, context): Promise<PaymentCollectionDTO>
deletePaymentCollections(ids[], context): Promise<void>

// PaymentSession
createPaymentSession(collectionId, data, context): Promise<PaymentSessionDTO>
updatePaymentSession(data, context): Promise<PaymentSessionDTO>
authorizePaymentSession(id, context, context): Promise<PaymentDTO>
deletePaymentSession(id, context): Promise<void>

// Payment
capturePayment(data, context): Promise<PaymentDTO>
refundPayment(data, context): Promise<PaymentDTO>
cancelPayment(id, context): Promise<PaymentDTO>

// Provider introspection
listPaymentProviders(filters, config, context): Promise<PaymentProviderDTO[]>
retrievePaymentSession(id, config, context): Promise<PaymentSessionDTO>

// AccountHolder
createAccountHolder(data, context): Promise<AccountHolderDTO>
updateAccountHolder(data, context): Promise<AccountHolderDTO>
deleteAccountHolders(ids[], context): Promise<void>

// Webhooks
processEvent(data: ProviderWebhookPayload, context): Promise<WebhookActionResult>
```

---

## Provider Interface

All payment providers implement `IPaymentProvider`:

```ts
interface IPaymentProvider {
  initiatePayment(data: InitiatePaymentInput): Promise<InitiatePaymentOutput>
  authorizePayment(data: AuthorizePaymentInput): Promise<AuthorizePaymentOutput>
  capturePayment(data: CapturePaymentInput): Promise<CapturePaymentOutput>
  refundPayment(data: RefundPaymentInput): Promise<RefundPaymentOutput>
  cancelPayment(data: CancelPaymentInput): Promise<CancelPaymentOutput>
  retrievePayment(data: RetrievePaymentInput): Promise<RetrievePaymentOutput>
  deletePayment(data: DeletePaymentInput): Promise<DeletePaymentOutput>
  getWebhookActionAndData(data: ProviderWebhookPayload): Promise<WebhookActionResult>
  // Account holder support (optional)
  createAccountHolder?(data): Promise<AccountHolderDTO>
  updateAccountHolder?(data): Promise<void>
  deleteAccountHolder?(data): Promise<void>
  listAccountHolderPaymentMethods?(data): Promise<PaymentMethodDTO[]>
}
```

---

## Module Dependencies

| Dependency | Direction | Reason |
|---|---|---|
| `@medusajs/framework/utils` | Required | `MedusaService`, BigNumber, enums |
| Payment providers | Plugin | Loaded as sub-providers via DI loader |
| `@medusajs/event-bus` | Optional, outbound | Emits payment lifecycle events |

The Payment module has **no dependency** on Cart or Order — it is associated via module links. Workflows pass `cart_id` / `order_id` as reference data stored in `PaymentCollection`.

---

## Key Workflows

| Workflow | Description |
|---|---|
| `createPaymentCollectionWorkflow` | Creates a `PaymentCollection` for a cart or order |
| `addPaymentSessionsWorkflow` | Initiates payment sessions for selected providers |
| `authorizePaymentSessionWorkflow` | Calls provider `authorizePayment`; creates `Payment` record |
| `capturePaymentWorkflow` | Calls provider `capturePayment`; creates `Capture` record |
| `refundPaymentWorkflow` | Calls provider `refundPayment`; creates `Refund` record |
| `cancelPaymentWorkflow` | Calls provider `cancelPayment`; updates payment status |
| `processPaymentWorkflow` | Handles inbound webhook; routes to capture/refund/cancel |

---

## Admin & Store API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/payment-collections` | List payment collections |
| `GET` | `/admin/payment-collections/:id` | Retrieve payment collection |
| `POST` | `/admin/payment-collections/:id` | Update payment collection |
| `DELETE` | `/admin/payment-collections/:id` | Delete payment collection |
| `GET` | `/admin/payments` | List payments |
| `POST` | `/admin/payments/:id/capture` | Capture payment |
| `POST` | `/admin/payments/:id/refund` | Refund payment |
| `POST` | `/store/payment-collections` | Create payment collection |
| `POST` | `/store/payment-collections/:id/payment-sessions` | Create / update payment session |
| `DELETE` | `/store/payment-collections/:id/payment-sessions/:session_id` | Delete payment session |
| `POST` | `/hooks/payment/:provider_id` | Webhook endpoint (per provider) |

---

## Configuration Options

```ts
// medusa-config.ts
{
  resolve: "@medusajs/payment",
  options: {
    providers: [
      {
        resolve: "@medusajs/payment-stripe",
        id: "stripe",
        options: {
          apiKey: process.env.STRIPE_API_KEY,
          webhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
        }
      }
    ]
  }
}
```

---

## Extension Points

| Extension | How |
|---|---|
| Custom provider | Implement `IPaymentProvider`; register in `options.providers` |
| Payment events | `payment.captured`, `payment.refunded`, `payment.authorized` |
| Webhook handling | `POST /hooks/payment/:provider_id` auto-routes to `processEvent()` |
| Saved payment methods | Implement `AccountHolder` methods in provider |
| Refund reasons | `createRefundReason()` service method |
