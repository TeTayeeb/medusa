# arc42 Architecture Document — Pricing Module

**Module:** `@medusajs/pricing`  
**Version:** Medusa v2.15.4  
**arc42 Template Version:** 8.x

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Pricing module must:
- Store price data decoupled from product/variant identity (linked via Module Links).
- Evaluate the best price for a given PriceSet given an arbitrary pricing context.
- Support time-bounded price lists (SALE overrides, OVERRIDE type).
- Support rule-based pricing (region, customer group, quantity tiers).
- Record per-currency, per-attribute tax-inclusion preferences.

### 1.2 Quality Goals
| Priority | Quality Goal | Motivation |
|---|---|---|
| 1 | **Calculation correctness** | Wrong prices directly impact revenue |
| 2 | **Query performance** | Price calculation runs on every product listing page load |
| 3 | **Extensibility** | New rule attributes must be addable without code changes |
| 4 | **Auditability** | Price list history must be traceable |

---

## 2. Architecture Constraints

- Prices must be stored as `BigNumber` (NUMERIC + raw JSONB) to avoid floating-point errors.
- The price-calculation query is a single optimised SQL query — no N+1 loops.
- The module must not import from the Product, Region, or Customer modules.
- Rule attributes are open strings (not an enum) to allow extension.

---

## 3. System Scope and Context

```
┌──────────────────────────────────────────────────────────┐
│                     Medusa Application                   │
│                                                          │
│  [Product Module] ──link── [Pricing Module] ←── [Cart]   │
│  (variant→price_set)              │          (checkout)  │
│                            [Region Module]               │
│                            (region_id rule attr)         │
│                            [Customer Module]             │
│                            (customer_group_id rule attr) │
└──────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Decision | Rationale |
|---|---|
| PriceSet as indirection layer | Variants and shipping options link to `PriceSet.id`, not to individual prices. This allows multi-currency and multi-rule prices without coupling to product schema. |
| Open rule attributes | `PriceRule.attribute` is a free-text field (e.g. `"region_id"`, `"customer_group_id"`). New dimensions are addable without migrations. |
| Ranking by `rules_count` | The most-specific match wins. Counting matched rules as a discriminator is simpler than a weighted scoring system. |
| Separate PriceList entity | Allows time-bounded and group-scoped override prices to be managed independently from base prices. |
| PricePreference for tax inclusion | Decouples tax-inclusive display logic from the price amount itself, supporting both tax-inclusive and tax-exclusive storefronts. |

---

## 5. Building Block View

### Level 1 — Module Structure

```
@medusajs/pricing
├── models/
│   ├── price-set.ts
│   ├── price.ts
│   ├── price-rule.ts
│   ├── price-list.ts
│   ├── price-list-rule.ts
│   └── price-preference.ts
├── repositories/
│   └── pricing.ts              ← Custom PricingRepositoryService
├── services/
│   └── pricing-module.ts       ← IPricingModuleService implementation
├── utils/
│   └── validate-price-list-dates.ts
├── joiner-config.ts
└── index.ts
```

### Level 2 — Calculation Pipeline

```
calculatePrices(filters, context)
  │
  ├─1─ Load candidate prices (price_set_id IN filters.id)
  ├─2─ Filter by currency_code match (if in context)
  ├─3─ Evaluate PriceRules against context attributes
  ├─4─ Filter active PriceLists (status=active, date range)
  ├─5─ Rank: rules_count DESC → list_type (OVERRIDE > SALE > base)
  └─6─ Return one CalculatedPriceSet per requested ID
```

---

## 6. Runtime View

### 6.1 Product Listing with Prices

```
GET /store/products
  → remote query (useQueryGraphStep)
      → fetches ProductVariant fields
      → joins PriceSet via link table
      → calls calculatePrices({id: [price_set_ids]}, { currency_code, region_id, customer_group_id })
  → returns variants with calculated_price inline
```

### 6.2 Add Prices to PriceSet

```
POST /admin/products/:id/variants/:variant_id  (with prices[])
  → createProductVariantsPriceSetsWorkflow
      → createPriceSetsStep (Pricing Module: createPriceSets)
      → createPricingRulesStep (Pricing Module: addPrices with rules)
      → createRemoteLinkStep (link variant_id → price_set_id)
```

### 6.3 Price List Activation

```
POST /admin/price-lists/:id  (status: "active")
  → updatePriceListsWorkflow
      → updatePriceLists (Pricing Module: updatePriceLists)
      → validatePriceListDates (util: ends_at > starts_at)
  → price_list.updated event emitted
```

---

## 7. Deployment View

The Pricing module runs in-process. The custom `PricingRepositoryService` executes against the shared PostgreSQL database. For high-traffic storefronts, the calculated price results can be cached at the application layer (e.g. Redis) with invalidation on `price_set.*` events.

---

## 8. Cross-Cutting Concepts

### 8.1 BigNumber Storage
Amount fields use a dual-column pattern: `amount NUMERIC` for DB arithmetic and `raw_amount JSONB` for lossless precision on read. The `BigNumber` utility class handles serialisation/deserialisation transparently.

### 8.2 Soft Deletes
All pricing entities support soft delete via `deleted_at`. Prices in deleted price lists are excluded from calculation automatically.

### 8.3 Tax Inclusion
`PricePreference.is_tax_inclusive` is resolved per `(attribute, value)` pair (e.g. `currency_code = EUR`) and returned alongside the calculated price for storefront display logic.

---

## 9. Architecture Decisions

### ADR-01: Open-Ended Rule Attributes
**Status:** Accepted  
**Context:** Rule dimensions (region, customer group, quantity) are not fixed — merchants want custom attributes.  
**Decision:** `PriceRule.attribute` is a free-text string validated at calculation time against a configurable set of known attributes.  
**Consequences:** Schema migration-free extension; risk of typos in attribute names.

### ADR-02: Custom Pricing Repository
**Status:** Accepted  
**Context:** Standard CRUD services cannot efficiently implement the ranking query.  
**Decision:** Introduce a dedicated `PricingRepositoryService` with a hand-crafted SQL `calculatePrices` method.  
**Consequences:** Higher performance at the cost of a custom repository that must be maintained.

### ADR-03: PriceList as Override Layer
**Status:** Accepted  
**Context:** Sale prices must co-exist with base prices and be toggleable.  
**Decision:** `PriceList` type OVERRIDE completely replaces base price; SALE applies only when lower.  
**Consequences:** Clear semantics; merchants can preview price list impact before activation.

---

## 10. Risks and Technical Debt

| Risk | Severity | Mitigation |
|---|---|---|
| Stale price cache | High | Invalidate on `price_set.*` and `price_list.*` events |
| Unbounded rule attributes | Medium | Validate attributes against known set at write time |
| N+1 in remote query expansion | Medium | `calculatePrices` accepts batched PriceSet IDs |
