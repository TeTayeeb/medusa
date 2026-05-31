# Promotion Module

**Package:** `@medusajs/promotion`  
**Version:** Medusa v2.15.4  
**Module Key:** `Modules.PROMOTION`

## Overview

The Promotion Module manages the full lifecycle of discounts, coupon codes, and promotional campaigns in a Medusa storefront. It provides a flexible rule engine that evaluates cart conditions and computes discount actions, supporting a broad spectrum of discount strategies — from simple percentage-off codes to complex buy-X-get-Y promotions scoped to specific line items or shipping methods.

The module is designed to operate independently of the Order and Cart modules. It exposes a well-defined service interface (`IPromotionModuleService`) consumed by core workflows, allowing discount logic to remain decoupled from cart assembly and order management.

## Key Concepts

### Promotions

A `Promotion` is the central entity. It carries a unique `code`, a `type` (standard or `buyget`), a `status` (draft, active, inactive), and optional `limit`/`used` counters for global usage caps. The `is_automatic` flag distinguishes promotions applied automatically on every cart from those requiring explicit coupon entry. The `is_tax_inclusive` flag controls whether the discount value already includes tax.

**Promotion types:**
| Type | Description |
|---|---|
| `standard` | Percentage, fixed-amount, or free-shipping discounts |
| `buyget` | Buy a qualifying quantity, get another item discounted |

**Promotion status:**
| Status | Description |
|---|---|
| `draft` | Not active; not applied to any cart |
| `active` | Currently applicable to carts |
| `inactive` | Disabled; previously active |

### Application Methods

Each promotion has exactly one `ApplicationMethod`. This entity defines *how* and *where* the discount is applied:

- **type:** `percentage`, `fixed`, `free_shipping`
- **target_type:** `order` (whole cart), `items` (specific line items), `shipping` (shipping methods)
- **allocation:** `each` (apply discount to each qualifying item) or `across` (split the discount across items)
- **value / currency_code:** The monetary or percentage value of the discount
- **max_quantity / apply_to_quantity:** Controls how many units receive the discount in buy-get scenarios
- **target_rules / buy_rules:** Many-to-many links to `PromotionRule` sets that further filter which items qualify

### Campaigns

Campaigns group multiple promotions under a shared time window and budget. A `Campaign` has:
- `campaign_identifier` — a unique business key for the campaign
- `starts_at` / `ends_at` — optional validity dates
- A single `CampaignBudget` with type `usage` (max number of redemptions) or `spend` (max monetary value discounted)

`CampaignBudget` tracks actual `used` totals and, since v2.11.0, can be scoped per-attribute (e.g., per `customer_id`), maintaining per-entity usage records in `CampaignBudgetUsage`.

### Rule Engine

`PromotionRule` entities represent conditional predicates:
- `attribute` — the cart attribute to test (e.g., `customer_group_id`, `region_id`, `cart_total`, `item_quantity`)
- `operator` — comparison operator: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`
- `values` — one or more `PromotionRuleValue` entries the attribute is compared against

Rules can be attached at two levels:
1. **Promotion-level rules** — determine whether the promotion is eligible for the whole cart
2. **ApplicationMethod rules** — `target_rules` (which items receive the discount) and `buy_rules` (which items must be purchased to qualify)

## Key Workflows

| Workflow | Description |
|---|---|
| `createPromotionWorkflow` | Creates a promotion with its application method and rules |
| `updatePromotionsWorkflow` | Updates promotion fields and rules |
| `deletePromotionsWorkflow` | Soft-deletes promotions with compensation |
| `createCampaignsWorkflow` | Creates a campaign with an optional budget |
| `computeActionsWorkflow` | Core engine: evaluates promotions against a cart and produces discount actions |
| `revertCouponCodesWorkflow` | Removes applied coupon codes and recalculates discounts |
| `registerUsageWorkflow` | Increments usage counters on campaign budgets after an order is placed |

## Admin API

| Endpoint | Description |
|---|---|
| `GET /admin/promotions` | List promotions |
| `POST /admin/promotions` | Create a promotion |
| `GET /admin/promotions/:id` | Get a promotion |
| `POST /admin/promotions/:id` | Update a promotion |
| `DELETE /admin/promotions/:id` | Delete a promotion |
| `GET /admin/campaigns` | List campaigns |
| `POST /admin/campaigns` | Create a campaign |
| `GET /admin/campaigns/:id` | Get a campaign |
| `POST /admin/campaigns/:id` | Update a campaign |
| `DELETE /admin/campaigns/:id` | Delete a campaign |

## Module Links

The Promotion Module integrates with other Medusa modules through declarative module links:

- **Cart ↔ Promotion** — links applied promotion codes to a cart
- **Order ↔ Promotion** — preserves discount data on placed orders for reporting and refund calculations

## Integration Points

- `@medusajs/cart` — provides cart totals and line items for rule evaluation
- `@medusajs/order` — records which promotions were applied to an order
- `@medusajs/region` — region rules reference region IDs
- `@medusajs/customer` — customer group rules reference customer group IDs

## Configuration

Register the module in `medusa-config.ts`:

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    { resolve: "@medusajs/promotion" }
  ]
})
```

The module uses MikroORM with PostgreSQL. All entities support soft-delete via `deleted_at`.
