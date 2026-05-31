# SpecKit — Pricing Module

**Module:** `@medusajs/pricing`  
**Version:** Medusa v2.15.4  
**Document type:** Functional & Technical Specification

---

## 1. Functional Specifications

### 1.1 PriceSets and Prices

**F-PRICE-01: Create PriceSet**
- A `PriceSet` is an empty container. Prices are added via `addPrices`.
- PriceSets are linked to product variants or shipping options via Module Links (not stored in the Pricing module).

**F-PRICE-02: Add Prices**
- Each `Price` within a `PriceSet` must have a `currency_code` and `amount` (BigNumber ≥ 0).
- A `Price` may have `min_quantity` and/or `max_quantity` for quantity-tiered pricing.
- A `Price` may have one or more `PriceRule` conditions.
- `rules_count` is incremented per rule on the `Price` record for ranking.

**F-PRICE-03: Delete PriceSet**
- Cascades to all child `Price` records and their `PriceRule` records.

### 1.2 Price Calculation

**F-CALC-01: calculatePrices Input**
- `filters.id` — array of PriceSet IDs to compute prices for.
- `context.currency_code` — required for currency filtering.
- `context.region_id` — optional, matched against `PriceRule.attribute = "region_id"`.
- `context.customer_group_id` — optional, matched against `PriceRule.attribute = "customer_group_id"`.
- `context.quantity` — optional, matched against `Price.min_quantity` / `max_quantity`.

**F-CALC-02: Matching Rules**
- A `Price` is a candidate only if **all** its `PriceRule` entries match the context.
- Rule operators: `eq` (default), `gt`, `gte`, `lt`, `lte`, `in`.
- A price with 0 rules (base price) matches any context.

**F-CALC-03: Price Selection Priority**
1. Prices from active `PriceList` with type `OVERRIDE` and highest `rules_count`.
2. Prices from active `PriceList` with type `SALE` and highest `rules_count`.
3. Base prices (no price_list) with highest `rules_count`.
4. Tie-break: lowest `amount`.

**F-CALC-04: PriceList Activation**
- `status` must be `active`.
- `now()` must be within `[starts_at, ends_at]` if set (null bounds are treated as open-ended).

### 1.3 PriceLists

**F-LIST-01: Create PriceList**
- `title` and `description` are required.
- `type` must be `SALE` or `OVERRIDE`.
- `status` defaults to `DRAFT`; must be explicitly set to `ACTIVE`.
- `starts_at` and `ends_at` are optional; if both set, `ends_at` must be after `starts_at`.

**F-LIST-02: PriceList Rules**
- A `PriceListRule` has `attribute` (string) and `value` (JSON array of acceptable values).
- All `PriceListRule` conditions on a list must match for the list to be considered.

### 1.4 Price Preferences

**F-PREF-01: Upsert Preference**
- `(attribute, value)` pair is unique.
- `is_tax_inclusive` defaults to `false`.
- Setting `is_tax_inclusive = true` for `attribute = "currency_code", value = "EUR"` means all EUR prices displayed to customers already include tax.

---

## 2. Business Rules

| ID | Rule | Enforcement |
|---|---|---|
| BR-01 | `amount` must be ≥ 0 | API-layer validation |
| BR-02 | `PriceRule (price_id, attribute, operator)` must be unique | DB partial unique index |
| BR-03 | `PricePreference (attribute, value)` must be unique | DB partial unique index |
| BR-04 | `ends_at > starts_at` when both provided | `validatePriceListDates` util |
| BR-05 | PriceSet delete cascades to Prices and PriceRules | DML cascade |
| BR-06 | PriceList delete cascades to PriceListRules and Prices in the list | DML cascade |
| BR-07 | `rules_count` on Price reflects number of attached PriceRules | Maintained by `addPrices` |

---

## 3. API Contracts

### 3.1 POST /admin/price-lists

**Request:**
```json
{
  "title": "Summer Sale",
  "description": "20% off all products",
  "type": "sale",
  "status": "active",
  "starts_at": "2024-06-01T00:00:00.000Z",
  "ends_at": "2024-08-31T23:59:59.000Z",
  "rules": [
    { "attribute": "customer_group_id", "value": ["cusgroup_vip"] }
  ],
  "prices": [
    {
      "currency_code": "usd",
      "amount": 8000,
      "variant_id": "variant_01"
    }
  ]
}
```

**Response `200`:**
```json
{
  "price_list": {
    "id": "plist_01EXAMPLE",
    "title": "Summer Sale",
    "status": "active",
    "type": "sale",
    "starts_at": "2024-06-01T00:00:00.000Z",
    "ends_at": "2024-08-31T23:59:59.000Z"
  }
}
```

**Errors:**
- `400 Bad Request` — `ends_at` is before `starts_at`
- `400 Bad Request` — invalid `type` value

### 3.2 GET /admin/price-preferences

**Query params:**
| Param | Type | Description |
|---|---|---|
| `attribute` | string | Filter by attribute (e.g. `currency_code`) |
| `value` | string | Filter by value (e.g. `EUR`) |

**Response `200`:**
```json
{
  "price_preferences": [
    {
      "id": "prpref_01",
      "attribute": "currency_code",
      "value": "EUR",
      "is_tax_inclusive": true
    }
  ],
  "count": 1
}
```

---

## 4. calculatePrices Response

### CalculatedPriceSet DTO
```ts
{
  id: string                          // PriceSet ID
  is_calculated_price_price_list: boolean
  is_calculated_price_tax_inclusive: boolean
  calculated_amount: number | null    // winning price amount
  raw_calculated_amount: object | null
  is_original_price_price_list: boolean
  is_original_price_tax_inclusive: boolean
  original_amount: number | null      // pre-list base price
  currency_code: string | null
  calculated_price: {
    id: string
    price_list_id: string | null
    price_list_type: string | null
    min_quantity: number | null
    max_quantity: number | null
  } | null
  original_price: { ... } | null
}
```

---

## 5. Validation Rules

| Field | Rule |
|---|---|
| `currency_code` | Lowercase ISO 4217 (3 chars) |
| `amount` | BigNumber ≥ 0 |
| `min_quantity` | BigNumber ≥ 0, must be ≤ `max_quantity` if both present |
| `operator` | One of: `eq`, `gt`, `gte`, `lt`, `lte`, `in` |
| `status` | One of: `draft`, `active` |
| `type` | One of: `sale`, `override` |
| `starts_at` / `ends_at` | ISO 8601 datetime; `ends_at > starts_at` |

---

## 6. Integration Test Scenarios

| Scenario | Expected |
|---|---|
| `calculatePrices` with no matching rules | Returns base price (0 rules) |
| Two prices with same currency, different rule counts | Higher rules_count wins |
| Active OVERRIDE list vs base price | OVERRIDE list price wins |
| PriceList outside date range | List prices excluded from calculation |
| `is_tax_inclusive = true` for EUR | Returned in `is_calculated_price_tax_inclusive` |
| Delete PriceSet | All child Prices and PriceRules deleted |
