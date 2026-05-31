# SpecKit — Order Module

> Medusa v2.15.4 | Functional & Technical Specification

---

## 1. Functional Capabilities

### 1.1 Order Placement
- Convert a completed cart into an order via `createOrderWorkflow`
- Snapshot all cart item details (price, title, quantity, variant/product IDs) onto order items — immune to future catalog changes
- Record shipping method selection with adjustments and tax lines

### 1.2 Order Lifecycle Management
- Cancel orders (with fulfilment and payment compensation)
- Complete orders (finalise financial summary)
- Mark order as `processing` after payment capture

### 1.3 Order Editing
- Open an order edit change set (draft state)
- Add, update, remove line items within the edit
- Preview financial delta (`calculateOrderChange`)
- Confirm or decline the edit

### 1.4 Return Management
- Create return request specifying items, quantities, reason
- Attach return shipping method
- Receive return items (update inventory)
- Cancel pending returns

### 1.5 Exchange Management
- Initiate exchange: return old items + add new outbound items
- Exchange creates an internal new order / outbound shipment

### 1.6 Claim Management
- Create warranty/defect claims with item images as evidence
- Support `refund` and `replace` resolution types
- Track claim items independently from order items

### 1.7 Financial Tracking
- Record `OrderTransaction` for every payment event (capture, refund, etc.)
- Maintain `OrderSummary` snapshot per order version
- Support store-credit `CreditLine` application

---

## 2. Business Rules

| Rule | Description |
|---|---|
| BR-O-001 | An order can only be cancelled if its status is `pending` or `processing` |
| BR-O-002 | Only one `active` (pending) `OrderChange` is allowed per order at a time |
| BR-O-003 | Return item quantity must not exceed original ordered quantity minus previously returned quantity |
| BR-O-004 | `Order.version` must be monotonically increasing; never decremented |
| BR-O-005 | An `OrderSummary` is immutable once written |
| BR-O-006 | All monetary amounts must be non-negative |
| BR-O-007 | `currency_code` must be a valid ISO 4217 code (normalised to lowercase) |
| BR-O-008 | A draft order (`is_draft_order = true`) cannot be completed without converting to a real order first |
| BR-O-009 | `ReturnReason` must reference a valid, non-deleted reason from the catalogue |
| BR-O-010 | Order edits cannot be confirmed if the order has been cancelled |

---

## 3. API Contracts

### GET `/admin/orders`

**Query Parameters**
```
id[]               Filter by IDs
status[]           Filter by status: pending, processing, shipped, delivered, cancelled, returned
customer_id
region_id
sales_channel_id[]
fulfillment_status[]
payment_status[]
q                  Full-text on email, display_id
created_at[gte]    Date range filter
created_at[lte]
limit              Default 20
offset             Default 0
fields             Field selector
```

**Response** `200 OK`
```ts
{
  orders: OrderDTO[]
  count: number
  offset: number
  limit: number
}
```

### POST `/admin/returns`

**Request**
```ts
{
  order_id: string                  // required
  items: {
    id: string                      // order item id
    quantity: number
    reason_id?: string
    note?: string
  }[]
  return_shipping?: {
    option_id: string
    price?: number
  }
  note?: string
  receive_now?: boolean             // immediately mark items as received
  no_notification?: boolean
  refund_amount?: number
  location_id?: string
  metadata?: Record<string, unknown>
}
```

**Response** `200 OK`
```ts
{ return: ReturnDTO }
```

### POST `/admin/orders/:id/cancel`

**Request**: No body required (optional `{ no_notification: boolean }`)

**Response** `200 OK`
```ts
{ order: OrderDTO }
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `order_id` | Must reference existing, non-deleted, non-cancelled order |
| `items[].quantity` | Positive integer; must not exceed returnable quantity |
| `items[].reason_id` | Must reference existing, non-deleted `ReturnReason` |
| `currency_code` | Valid ISO 4217; normalised to lowercase |
| `amount` (transaction) | Non-negative BigNumber |
| `version` | Read-only; managed internally |
| `status` transitions | Enforced by service guard methods |
| `display_id` | Auto-increment; read-only |
| `custom_display_id` | Unique (partial index); max 255 chars |

---

## 5. Performance Considerations

| Concern | Mitigation |
|---|---|
| Order list queries with many joins | Selective relation loading; `fields` parameter limits joined tables |
| Total computation on large orders | `OrderSummary` caches computed totals per version — O(1) read |
| BigNumber arithmetic overhead | `MathBN` uses string-based `decimal.js`; acceptable for item counts ≤ 10k |
| Return / exchange history | Separate entities; lazy-loaded unless explicitly requested in `fields` |
| Audit log growth | `OrderChangeAction` rows accumulate; archive strategy recommended after 90 days |
| Index coverage | Partial indexes on `status`, `customer_id`, `region_id`, `sales_channel_id`, `deleted_at` |
