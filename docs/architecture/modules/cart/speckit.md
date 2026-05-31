# SpecKit — Cart Module

> Medusa v2.15.4 | Functional & Technical Specification

---

## 1. Functional Capabilities

### 1.1 Cart Management
- Create a cart with optional customer, region, sales channel, currency, email
- Retrieve a cart with fully computed totals
- Update cart-level data (email, region, customer, addresses, metadata)
- Mark cart as completed on successful checkout

### 1.2 Line Item Management
- Add product variant line items with quantity and price snapshot
- Update line item quantity (minimum 1)
- Remove line items individually or in bulk
- Store subtitle, thumbnail, variant/product IDs as display snapshot

### 1.3 Discount & Promotion Adjustments
- Apply line item adjustments with `promotion_id` reference and amount
- Apply shipping method adjustments
- Replace entire adjustment set atomically (set-semantics)
- Remove individual adjustments

### 1.4 Tax Line Management
- Apply per-item tax lines (rate, rate_id, provider_id)
- Apply per-shipping-method tax lines
- Replace tax line sets atomically

### 1.5 Shipping Method Selection
- Add shipping methods with amount and metadata
- Remove shipping methods
- Support multiple shipping methods per cart (multi-fulfillment)

### 1.6 Store Credit / Gift Cards
- Apply `CreditLine` records representing store credit or gift card balances
- Multiple credit lines allowed per cart
- Credit lines excluded from cart cascade deletion

### 1.7 Checkout Completion
- Validate cart readiness (address, payment session)
- Trigger payment authorization
- Convert cart to order
- Mark cart `completed_at`

---

## 2. Business Rules

| Rule | Description |
|---|---|
| BR-C-001 | `currency_code` must be a valid ISO 4217 code (normalised to lowercase) |
| BR-C-002 | Line item quantity must be a positive integer ≥ 1 |
| BR-C-003 | A cart with `completed_at` set cannot accept further mutations |
| BR-C-004 | `unit_price` on a line item must be non-negative |
| BR-C-005 | `setLineItemAdjustments()` replaces ALL adjustments for the cart atomically |
| BR-C-006 | A cart must have a `shipping_address` before `completeCartWorkflow` can run |
| BR-C-007 | Each line item must reference a valid `variant_id` (validated at workflow level, not module level) |
| BR-C-008 | `CreditLine.amount` must be non-negative |
| BR-C-009 | Cart total after adjustments and credits must be ≥ 0 before checkout |
| BR-C-010 | Multiple shipping methods are allowed; totals aggregate across all |

---

## 3. API Contracts

### POST `/store/carts`

**Request**
```ts
{
  region_id?: string
  sales_channel_id?: string
  email?: string
  currency_code?: string
  shipping_address?: AddressInput
  billing_address?: AddressInput
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ cart: CartDTO }
```

### POST `/store/carts/:id/line-items`

**Request**
```ts
{
  variant_id: string     // required
  quantity: number       // required, positive integer
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ cart: CartDTO }        // full cart with updated totals
```

### POST `/store/carts/:id` (Update Cart)

**Request**
```ts
{
  region_id?: string
  email?: string
  shipping_address?: AddressInput
  billing_address?: AddressInput
  sales_channel_id?: string
  promo_codes?: string[]     // apply/remove promotion codes
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ cart: CartDTO }
```

### POST `/store/carts/:id/complete`

**Request**: No body required

**Response** `200 OK`
```ts
{
  type: "order" | "cart"    // "cart" if payment needs more action
  order?: OrderDTO
  cart?: CartDTO
}
```

### CartDTO Shape (key fields)

```ts
{
  id: string
  region_id: string | null
  customer_id: string | null
  email: string | null
  currency_code: string
  completed_at: string | null
  shipping_address: CartAddressDTO | null
  billing_address: CartAddressDTO | null
  items: CartLineItemDTO[]
  shipping_methods: CartShippingMethodDTO[]
  // Computed totals (all BigNumber):
  total: number
  subtotal: number
  tax_total: number
  discount_total: number
  shipping_total: number
  item_total: number
  gift_card_total: number
  // ... all original_* variants
}
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `currency_code` | Valid ISO 4217; normalised to lowercase at service layer |
| `quantity` (line item) | Integer ≥ 1 |
| `unit_price` | Non-negative BigNumber |
| `amount` (adjustment) | BigNumber; can be negative (discount); must not exceed item total |
| `amount` (shipping method) | Non-negative BigNumber |
| `email` | Valid email format (validated at API route level via `validateEmail()`) |
| `region_id` | Must reference existing region (validated at workflow level) |
| `variant_id` | Must reference existing, non-deleted variant (validated at workflow level) |

---

## 5. Performance Considerations

| Concern | Mitigation |
|---|---|
| Total recomputation cost | `decorateCartTotals()` is O(n items × m adjustments); acceptable for ≤ 50 items |
| `retrieveCart` latency | Single query with explicit joins; avoids N+1 via `FindConfig.relations` |
| Adjustment replacement (set-semantics) | Single DELETE + bulk INSERT in one transaction; no individual row diffs |
| Cart abandonment cleanup | Background job recommended; carts with `completed_at IS NULL` older than TTL |
| Concurrent item updates | Row-level locking via MikroORM transaction manager; last write wins |
| Index coverage | Partial indexes on `region_id`, `customer_id`, `sales_channel_id`, `currency_code` |
| Large cart serialisation | `fields` parameter limits response payload; avoid eagerly loading all relations |
