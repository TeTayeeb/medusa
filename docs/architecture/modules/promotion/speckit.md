# SpecKit — Promotion Module

**Module:** `@medusajs/promotion`  
**Version:** Medusa v2.15.4  
**Document Type:** Functional & Technical Specification

---

## 1. Functional Requirements

### FR-1: Promotion Management

| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | System SHALL support creating promotions with a unique code, type (standard/buyget), and status (draft/active/inactive) | MUST |
| FR-1.2 | System SHALL prevent duplicate promotion codes among non-deleted promotions | MUST |
| FR-1.3 | System SHALL support global redemption limits (`limit` field) per promotion | MUST |
| FR-1.4 | System SHALL track global usage count (`used` field) per promotion | MUST |
| FR-1.5 | System SHALL allow promotions to be marked as automatic (no code required) | MUST |
| FR-1.6 | System SHALL support tax-inclusive discount values | SHOULD |
| FR-1.7 | System SHALL support metadata on promotions for extensibility | SHOULD |

### FR-2: Application Method

| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | Each promotion SHALL have exactly one application method | MUST |
| FR-2.2 | Application method type SHALL be one of: `percentage`, `fixed`, `free_shipping` | MUST |
| FR-2.3 | Target type SHALL be one of: `order`, `items`, `shipping` | MUST |
| FR-2.4 | Allocation SHALL support `each` (per qualifying item) and `across` (split across items) | MUST |
| FR-2.5 | Buy-get promotions SHALL support `buy_rules_min_quantity`, `apply_to_quantity`, and `max_quantity` | MUST |
| FR-2.6 | Target rules SHALL filter which items receive the discount | MUST |
| FR-2.7 | Buy rules SHALL filter which items must be purchased to qualify | MUST |

### FR-3: Campaigns

| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | System SHALL support grouping promotions under campaigns with a unique identifier | MUST |
| FR-3.2 | Campaigns SHALL support optional start and end date/time | MUST |
| FR-3.3 | Campaigns SHALL support budget limits of type `usage` (count) or `spend` (monetary) | MUST |
| FR-3.4 | System SHALL track actual budget usage in real-time | MUST |
| FR-3.5 | System SHALL support per-attribute budget limits (e.g., per customer) | SHOULD |

### FR-4: Rule Engine

| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | Rules SHALL support attributes: `customer_group_id`, `region_id`, `cart_total`, `item_quantity`, `product_id`, `product_category_id`, `sku` | MUST |
| FR-4.2 | Rules SHALL support operators: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin` | MUST |
| FR-4.3 | Rules SHALL support multiple values (IN/NIN comparison) | MUST |
| FR-4.4 | Multiple rules on a promotion SHALL be evaluated with AND logic | MUST |
| FR-4.5 | Rule attributes SHALL be extensible without schema changes | SHOULD |

### FR-5: Discount Computation

| ID | Requirement | Priority |
|---|---|---|
| FR-5.1 | `computeActions` SHALL evaluate all active automatic promotions plus provided codes | MUST |
| FR-5.2 | `computeActions` SHALL return per-line-item discount adjustment actions | MUST |
| FR-5.3 | System SHALL enforce campaign date range validity during computation | MUST |
| FR-5.4 | System SHALL not exceed promotion `limit` or campaign budget in `registerUsage` | MUST |
| FR-5.5 | `computeActions` SHALL be idempotent and stateless | MUST |

---

## 2. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | `computeActions` response time | < 50ms for cart with ≤20 items and ≤5 promotions |
| NFR-2 | Unique code enforcement | Sub-millisecond via partial unique DB index |
| NFR-3 | Concurrent budget updates | Consistent eventual counter; over-redemption risk < 0.01% at moderate load |
| NFR-4 | API pagination | All list endpoints support cursor-based pagination |
| NFR-5 | Soft-delete coverage | 100% of entities; no hard deletes in standard flows |

---

## 3. API Specification

### POST /admin/promotions — Create Promotion

**Request Body:**
```json
{
  "code": "SUMMER20",
  "type": "standard",
  "status": "active",
  "is_automatic": false,
  "is_tax_inclusive": false,
  "limit": 100,
  "application_method": {
    "type": "percentage",
    "target_type": "items",
    "allocation": "each",
    "value": 20,
    "target_rules": [
      {
        "attribute": "product_category_id",
        "operator": "in",
        "values": [{ "value": "pcat_apparel" }]
      }
    ]
  },
  "rules": [
    {
      "attribute": "customer_group_id",
      "operator": "in",
      "values": [{ "value": "custgrp_vip" }]
    }
  ]
}
```

**Response:** `201 Created` with `PromotionDTO`

### POST /admin/campaigns — Create Campaign

**Request Body:**
```json
{
  "name": "Summer Sale 2025",
  "campaign_identifier": "summer-2025",
  "starts_at": "2025-06-01T00:00:00Z",
  "ends_at": "2025-08-31T23:59:59Z",
  "budget": {
    "type": "spend",
    "currency_code": "usd",
    "limit": 500000
  }
}
```

---

## 4. Workflow Specifications

### computeActionsWorkflow

**Input:**
```typescript
{
  promotion_codes: string[],
  cart: {
    id: string,
    items: CartLineItem[],
    shipping_methods: ShippingMethod[],
    customer_id?: string,
    customer_groups?: string[],
    region_id?: string,
    currency_code: string
  }
}
```

**Output:**
```typescript
ComputeActionAdjustmentLine[] — [
  {
    action: "addItemAdjustment" | "addShippingMethodAdjustment" | "removeItemAdjustment",
    item_id?: string,
    shipping_method_id?: string,
    amount: number,
    code?: string,
    promotion_id: string
  }
]
```

**Compensation:** None (read-only operation)

---

## 5. Data Validation Rules

| Field | Rule |
|---|---|
| `Promotion.code` | Non-empty string; unique; case-preserved |
| `Promotion.type` | Must be `standard` or `buyget` |
| `Promotion.status` | Must be `draft`, `active`, or `inactive` |
| `ApplicationMethod.value` | Must be positive number; required for `percentage` and `fixed` types |
| `ApplicationMethod.allocation` | Required when `target_type = items` |
| `CampaignBudget.currency_code` | Required when `type = spend` |
| `PromotionRule.operator` | Must be one of the 8 supported operators |
| `Campaign.ends_at` | Must be after `starts_at` if both are provided |

---

## 6. Error Conditions

| Error | Type | HTTP Status |
|---|---|---|
| Promotion code already exists | `INVALID_DATA` | 422 |
| Campaign identifier already exists | `INVALID_DATA` | 422 |
| Promotion not found | `NOT_FOUND` | 404 |
| Campaign not found | `NOT_FOUND` | 404 |
| Cannot activate a promotion without an application method | `INVALID_DATA` | 400 |
| Budget limit type requires currency_code | `INVALID_DATA` | 400 |
