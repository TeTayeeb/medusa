# SpecKit — Payment Module

> Medusa v2.15.4 | Functional & Technical Specification

---

## 1. Functional Capabilities

### 1.1 Payment Collection Management
- Create a `PaymentCollection` linked to a cart or order with a specific amount and currency
- Update collection amount (when order total changes after cart completion)
- Delete collection (cascades to sessions and payments)
- Mark collection as complete

### 1.2 Payment Session Management
- Initiate payment sessions for one or more providers simultaneously
- Update session data (e.g. after 3DS step)
- Delete individual sessions (customer changes mind)
- Re-use existing sessions on cart update

### 1.3 Payment Authorization
- Authorize a payment session via provider's `authorizePayment()` call
- Handle `requires_more` status (e.g. 3DS redirect, pending verification)
- Create `Payment` record on successful authorization

### 1.4 Capture
- Full or partial capture against an authorized payment
- Multiple captures against a single payment (up to authorized amount)
- Track remaining capturable amount

### 1.5 Refund
- Full or partial refund against a captured payment
- Multiple refunds up to total captured amount
- Attach `RefundReason` to each refund

### 1.6 Cancellation
- Cancel an uncaptured payment
- Cancel a payment session

### 1.7 Webhook Processing
- Receive provider webhooks at `/hooks/payment/:provider_id`
- Parse and route to correct action (capture, refund, authorize, fail)

### 1.8 Account Holders (Saved Payment Methods)
- Create account holder linking customer to provider customer record
- List saved payment methods
- Delete account holder

---

## 2. Business Rules

| Rule | Description |
|---|---|
| BR-PAY-001 | Only one payment session per provider per collection is allowed |
| BR-PAY-002 | A `PaymentSession` can only be authorized if its status is `pending` or `requires_more` |
| BR-PAY-003 | Total captured amount across all captures must not exceed `Payment.amount` |
| BR-PAY-004 | Total refunded amount across all refunds must not exceed captured amount |
| BR-PAY-005 | A canceled payment cannot be captured or refunded |
| BR-PAY-006 | `PaymentCollection.amount` must equal sum of all amounts expected from the order |
| BR-PAY-007 | `currency_code` must be a valid ISO 4217 code (normalised to lowercase) |
| BR-PAY-008 | All monetary amounts must be non-negative |
| BR-PAY-009 | Webhook processing must verify provider signature before acting |
| BR-PAY-010 | `AccountHolder` is unique per `(customer_id, provider_id)` pair |

---

## 3. API Contracts

### POST `/store/payment-collections`

**Request**
```ts
{
  cart_id: string       // required; amount derived from cart total
}
```

**Response** `200 OK`
```ts
{ payment_collection: PaymentCollectionDTO }
```

### POST `/store/payment-collections/:id/payment-sessions`

**Request**
```ts
{
  provider_id: string          // e.g. "pp_stripe_stripe"
  data?: Record<string, unknown>   // provider-specific session data
}
```

**Response** `200 OK`
```ts
{ payment_collection: PaymentCollectionDTO }
```

### POST `/admin/payments/:id/capture`

**Request**
```ts
{
  amount?: number    // partial capture; defaults to full remaining
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ payment: PaymentDTO }
```

### POST `/admin/payments/:id/refund`

**Request**
```ts
{
  amount: number                 // required
  reason_id?: string             // RefundReason id
  note?: string
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ payment: PaymentDTO }
```

### PaymentCollectionDTO Shape

```ts
{
  id: string
  currency_code: string
  amount: number
  authorized_amount: number | null
  captured_amount: number | null
  refunded_amount: number | null
  completed_at: string | null
  status: "not_paid" | "awaiting" | "authorized" | "partially_captured"
          | "captured" | "partially_refunded" | "refunded" | "canceled"
  payment_providers: PaymentProviderDTO[]
  payment_sessions: PaymentSessionDTO[]
  payments: PaymentDTO[]
}
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `currency_code` | Valid ISO 4217; normalised to lowercase |
| `amount` | Non-negative BigNumber; validated against `defaultCurrencies` precision |
| `provider_id` | Must reference a registered, active payment provider |
| `capture amount` | Non-negative; must not exceed `Payment.amount - SUM(prior captures)` |
| `refund amount` | Non-negative; must not exceed `SUM(captures) - SUM(prior refunds)` |
| `reason_id` | Must reference existing, non-deleted `RefundReason` |
| Payment session status | Must be `pending` or `requires_more` to authorize |
| Payment status | Must not be `canceled` to capture or refund |

---

## 5. Performance Considerations

| Concern | Mitigation |
|---|---|
| Provider API latency | All provider calls are async; workflow steps have configurable timeouts |
| Amount aggregation | `captured_amount` / `refunded_amount` recomputed on each capture/refund; acceptable for typical order sizes |
| Webhook throughput | Webhook endpoint is a lightweight route; heavy processing runs in background workflow |
| Provider registry lookup | In-memory `Map<providerId, IPaymentProvider>`; O(1) lookup |
| Currency precision table | `defaultCurrencies` is a compile-time constant object; O(1) lookup |
| Index coverage | Partial indexes on `provider_id`, `payment_collection_id`, `payment_session_id` on Payment |
| Capture/refund audit queries | Direct DB queries against `captures`/`refunds` tables; no aggregation cache needed for typical volumes |
