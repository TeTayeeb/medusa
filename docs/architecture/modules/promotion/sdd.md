# Software Design Document — Promotion Module

**Module:** `@medusajs/promotion`  
**Version:** Medusa v2.15.4  
**Status:** Production  
**Last Updated:** 2025

---

## 1. Purpose and Scope

This document describes the internal design, data model, service layer, and computational model of the Promotion Module. The module is responsible for evaluating promotional eligibility, computing discount amounts, and maintaining campaign budget state within the Medusa commerce platform.

---

## 2. System Context

The Promotion Module sits between the Cart domain and the Order domain. It receives a representation of the current cart (line items, totals, customer context) from upstream workflows, evaluates applicable promotions against the rule engine, and emits a list of discount actions back to the cart calculation pipeline. It writes no data to the cart directly.

```
Cart Module → computeActionsWorkflow → Promotion Module → Discount Actions → Cart Module
                                                        ↘ registerUsageWorkflow → Order placed
```

---

## 3. Data Model

### 3.1 Entity Relationship Summary

| Entity | Table | PK Prefix | Notes |
|---|---|---|---|
| `Promotion` | `promotion` | `promo_` | Core discount entity |
| `Campaign` | `promotion_campaign` | `procamp_` | Groups promotions |
| `CampaignBudget` | `promotion_campaign_budget` | `probudg_` | Tracks spend/usage limits |
| `CampaignBudgetUsage` | `promotion_campaign_budget_usage` | — | Per-attribute usage tracking |
| `ApplicationMethod` | `promotion_application_method` | `proappmet_` | How discount is applied |
| `PromotionRule` | `promotion_rule` | `prorul_` | Condition predicate |
| `PromotionRuleValue` | `promotion_rule_value` | `prorulval_` | Values for rule comparison |

### 3.2 Promotion Fields

```
Promotion {
  id            : string (PK, prefix: promo)
  code          : string (unique, case-sensitive, searchable)
  is_automatic  : boolean (default: false)
  is_tax_inclusive : boolean (default: false)
  limit         : number | null   -- global redemption cap (since v2.12)
  used          : number (default: 0)  -- global redemption count
  type          : PromotionType (standard | buyget)
  status        : PromotionStatus (draft | active | inactive)
  metadata      : json | null
  campaign_id   : FK → Campaign (nullable)
  deleted_at    : datetime | null
}
```

**Indexes:**
- `IDX_unique_promotion_code` — unique on `code` where `deleted_at IS NULL`
- `IDX_promotion_is_automatic` — supports efficient lookup of auto-apply promotions
- `IDX_promotion_type`, `IDX_promotion_status`

### 3.3 Campaign Fields

```
Campaign {
  id                   : string (PK, prefix: procamp)
  name                 : string (searchable)
  description          : string | null (searchable)
  campaign_identifier  : string (unique where deleted_at IS NULL)
  starts_at            : datetime | null
  ends_at              : datetime | null
  deleted_at           : datetime | null
}
```

Cascade delete: `budget`, and by extension `CampaignBudgetUsage` records.

### 3.4 CampaignBudget Fields

```
CampaignBudget {
  id            : string (PK, prefix: probudg)
  type          : CampaignBudgetType (usage | spend)
  currency_code : string | null   -- required for spend type
  limit         : bigNumber | null
  used          : bigNumber (default: 0)
  attribute     : string | null   -- e.g. "customer_id" (since v2.11)
  deleted_at    : datetime | null
}
```

When `attribute` is set, per-entity usage is recorded in `CampaignBudgetUsage` keyed by `(budget_id, attribute_value)`. The module checks per-entity usage before allowing redemption.

### 3.5 ApplicationMethod Fields

```
ApplicationMethod {
  id                   : string (PK, prefix: proappmet)
  type                 : ApplicationMethodType (percentage | fixed | free_shipping)
  target_type          : ApplicationMethodTargetType (order | items | shipping)
  allocation           : ApplicationMethodAllocation | null (each | across)
  value                : bigNumber | null
  currency_code        : string | null
  max_quantity         : number | null
  apply_to_quantity    : number | null
  buy_rules_min_quantity : number | null
  deleted_at           : datetime | null
}
```

**Pivot tables:**
- `application_method_target_rules (application_method_id, promotion_rule_id)` — item-level filter for discount target
- `application_method_buy_rules (application_method_id, promotion_rule_id)` — item-level qualification for buy-get promotions

### 3.6 PromotionRule Fields

```
PromotionRule {
  id          : string (PK, prefix: prorul)
  description : string | null
  attribute   : string (indexed)
  operator    : PromotionRuleOperator (eq | ne | gt | gte | lt | lte | in | nin)
  deleted_at  : datetime | null
}
```

**Pivot tables:**
- `promotion_promotion_rule (promotion_id, promotion_rule_id)` — promotion-level eligibility rules

Cascade delete: `values` (PromotionRuleValue).

---

## 4. Service Layer

### 4.1 IPromotionModuleService

The public service interface exposes CRUD operations for all entities and the core promotion computation logic:

```typescript
interface IPromotionModuleService {
  // Promotion CRUD
  createPromotions(data, context?): Promise<PromotionDTO[]>
  updatePromotions(data, context?): Promise<PromotionDTO[]>
  deletePromotions(ids, context?): Promise<void>
  softDeletePromotions(ids, config?, context?): Promise<void>
  listPromotions(filters?, config?, context?): Promise<PromotionDTO[]>
  retrievePromotion(id, config?, context?): Promise<PromotionDTO>

  // Campaign CRUD
  createCampaigns(data, context?): Promise<CampaignDTO[]>
  updateCampaigns(data, context?): Promise<CampaignDTO[]>
  deleteCampaigns(ids, context?): Promise<void>

  // Rule management
  addPromotionRules(promotionId, rules, context?): Promise<PromotionRuleDTO[]>
  removePromotionRules(promotionId, ruleIds, context?): Promise<void>
  addPromotionTargetRules(methodId, rules, context?): Promise<PromotionRuleDTO[]>
  addPromotionBuyRules(methodId, rules, context?): Promise<PromotionRuleDTO[]>

  // Core computation
  computeActions(codes, cartData, context?): Promise<ComputeActionAdjustmentLine[]>
  registerUsage(actions, context?): Promise<void>
}
```

### 4.2 computeActions Algorithm

1. Resolve `Promotion` records for the provided codes plus all `is_automatic = true` active promotions
2. Filter promotions by campaign date ranges (`starts_at`, `ends_at`)
3. Evaluate **promotion-level rules** against the cart context — discard non-matching promotions
4. For each passing promotion, delegate to the `ApplicationMethod` evaluator:
   - Evaluate `buy_rules` (for `buyget` type) — check qualifying items and quantity
   - Evaluate `target_rules` — filter line items or shipping methods to apply discount to
   - Calculate discount amount based on `type` (percentage/fixed/free_shipping) and `allocation` (each/across)
5. Return an array of `ComputeActionAdjustmentLine` objects (one per affected line item / shipping method)

### 4.3 registerUsage

After a successful order placement, `registerUsage` is called with the computed actions:
- Increments `Promotion.used` counter
- Increments `CampaignBudget.used` (total spend or usage)
- Creates/updates `CampaignBudgetUsage` records when per-attribute tracking is enabled

---

## 5. Soft-Delete Strategy

All entities implement soft-delete via a `deleted_at` timestamp column. Unique indexes include a `WHERE deleted_at IS NULL` predicate, allowing reuse of codes after soft-deletion. Hard-delete is not used in production flows.

---

## 6. Cascade Strategy

| Parent | Cascaded Children |
|---|---|
| `Promotion` | `application_method` |
| `Campaign` | `budget` → `usages` |
| `PromotionRule` | `values` |
| `ApplicationMethod` | (none; rules are many-to-many) |

---

## 7. Known Constraints & Limitations

- Promotion `code` must be globally unique among non-deleted promotions
- Campaign `campaign_identifier` must be globally unique among non-deleted campaigns
- Service zone names must be unique — shared rule applies at DB level
- `computeActions` is a read-only operation; it does not persist any state
- Budget decrement on usage is not atomic by default; high-concurrency scenarios require external locking or event sourcing patterns
