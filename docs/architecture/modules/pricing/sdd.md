# Software Design Document — Pricing Module

**Module:** `@medusajs/pricing`  
**Version:** Medusa v2.15.4  
**Status:** Stable

---

## 1. Purpose and Scope

This SDD describes the internal design of the Pricing module: its data model, service class hierarchy, the price-calculation algorithm, the custom pricing repository, and domain event strategy.

---

## 2. Data Model

### 2.1 Entity Relationship

```
PriceSet ──────────── Price ──────────── PriceRule
                        │
                      PriceList ──────── PriceListRule

PricePreference  (standalone, keyed by attribute+value)
```

### 2.2 Entity Definitions

#### PriceSet
```sql
CREATE TABLE price_set (
  id         TEXT PRIMARY KEY,   -- prefix: pset_
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
```

#### Price
```sql
CREATE TABLE price (
  id            TEXT PRIMARY KEY,  -- prefix: price_
  title         TEXT,
  currency_code TEXT NOT NULL,
  amount        NUMERIC NOT NULL,
  raw_amount    JSONB,
  min_quantity  NUMERIC,
  max_quantity  NUMERIC,
  rules_count   INTEGER DEFAULT 0,
  price_set_id  TEXT REFERENCES price_set(id),
  price_list_id TEXT REFERENCES price_list(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ
);
CREATE INDEX ON price (price_set_id) WHERE deleted_at IS NULL;
CREATE INDEX ON price (price_list_id) WHERE deleted_at IS NULL;
```

#### PriceRule
```sql
CREATE TABLE price_rule (
  id         TEXT PRIMARY KEY,  -- prefix: prule_
  attribute  TEXT NOT NULL,
  value      TEXT NOT NULL,
  operator   TEXT NOT NULL DEFAULT 'eq',  -- eq | gt | gte | lt | lte | in
  priority   INTEGER DEFAULT 0,
  price_id   TEXT REFERENCES price(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE UNIQUE INDEX ON price_rule (price_id, attribute, operator) WHERE deleted_at IS NULL;
```

#### PriceList
```sql
CREATE TABLE price_list (
  id          TEXT PRIMARY KEY,  -- prefix: plist_
  title       TEXT NOT NULL,
  description TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'draft',  -- draft | active
  type        TEXT NOT NULL DEFAULT 'sale',   -- sale | override
  starts_at   TIMESTAMPTZ,
  ends_at     TIMESTAMPTZ,
  rules_count INTEGER DEFAULT 0,
  metadata    JSONB,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at  TIMESTAMPTZ
);
CREATE INDEX ON price_list (id, status, starts_at, ends_at)
  WHERE deleted_at IS NULL AND status = 'active';
```

#### PriceListRule
```sql
CREATE TABLE price_list_rule (
  id            TEXT PRIMARY KEY,  -- prefix: prule_
  attribute     TEXT NOT NULL,
  value         JSONB,
  price_list_id TEXT REFERENCES price_list(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ
);
```

#### PricePreference
```sql
CREATE TABLE price_preference (
  id               TEXT PRIMARY KEY,  -- prefix: prpref_
  attribute        TEXT NOT NULL,
  value            TEXT,
  is_tax_inclusive BOOLEAN DEFAULT FALSE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at       TIMESTAMPTZ
);
CREATE UNIQUE INDEX IDX_price_preference_attribute_value
  ON price_preference (attribute, value) WHERE deleted_at IS NULL;
```

---

## 3. Service Architecture

### 3.1 Class Hierarchy

```
IPricingModuleService (interface from @medusajs/framework/types)
  └── PricingModuleService extends MedusaService<{...}>
        ├── pricingRepository_      (PricingRepositoryService — custom)
        ├── priceSetService_        (IMedusaInternalService<PriceSet>)
        ├── priceService_           (IMedusaInternalService<Price>)
        ├── priceRuleService_       (IMedusaInternalService<PriceRule>)
        ├── priceListService_       (IMedusaInternalService<PriceList>)
        ├── priceListRuleService_   (IMedusaInternalService<PriceListRule>)
        └── pricePreferenceService_ (IMedusaInternalService<PricePreference>)
```

### 3.2 Custom Pricing Repository

The module includes a `PricingRepositoryService` (`pricingRepository_`) beyond the standard CRUD services. This repository contains the **price-calculation query** — a performance-critical SQL query that:

1. Joins `price_set`, `price`, and `price_rule` in a single query.
2. Filters by active `price_list` date ranges.
3. Ranks candidates by `rules_count DESC` and `price_list.type` priority (OVERRIDE > SALE > base).
4. Returns the winning `Price` per `PriceSet`.

The repository also manages a cache of available rule `attribute` names for validation (`clearAvailableAttributes` / `setupCalculatedPriceConfig_`).

### 3.3 addPrices Flow

```
addPrices(AddPricesDTO)
  → for each price in dto.prices:
      1. Validate currency_code exists
      2. Create Price record
      3. Create PriceRule records for each rule in price.rules
      4. Increment rules_count on Price
  → return updated PriceSet
```

### 3.4 calculatePrices Flow

```
calculatePrices(filters, context)
  → pricingRepository_.calculatePrices(filters, context)
      1. Fetch candidate prices by price_set_id ∈ filters.id
      2. Evaluate price_rules against context
      3. Filter active price_lists (status=active, now() BETWEEN starts_at AND ends_at)
      4. Rank: rules_count DESC → list_type priority → amount ASC
      5. Return one CalculatedPriceSet per requested PriceSet ID
```

---

## 4. Repository Layer

Standard CRUD for all entities uses `IMedusaInternalService` generated bases. The custom `PricingRepositoryService` extends a base DAL repository with a hand-crafted `calculatePrices` method implementing the ranking logic described in §3.3.

---

## 5. Joiner Configuration

```ts
{
  serviceName: Modules.PRICING,
  alias: [
    { name: "price_set", args: { entity: "PriceSet" } },
    { name: "price_list", args: { entity: "PriceList" } },
    { name: "price_preference", args: { entity: "PricePreference" } },
  ],
  primaryKeys: ["id"],
  linkableKeys: { price_set_id: "PriceSet" },
}
```

---

## 6. Domain Events

| Method | Event |
|---|---|
| `createPriceSets` | `price_set.created` |
| `deletePriceSets` | `price_set.deleted` |
| `createPriceLists` | `price_list.created` |
| `updatePriceLists` | `price_list.updated` |
| `deletePriceLists` | `price_list.deleted` |

---

## 7. Validation

- `PricePreference (attribute, value)` uniqueness enforced at DB level.
- `PriceRule (price_id, attribute, operator)` uniqueness enforced at DB level.
- `PriceList` date range: `ends_at` must be after `starts_at` when both are provided (validated in `validatePriceListDates` utility).
- `currency_code` must match an active currency in the Currency module (validated in workflow steps, not the service itself).

---

## 8. Migrations

| Migration | Change |
|---|---|
| `20230929122253` | Initial schema |
| `20240322094407` | Rule operator support |
| `20240626133555` | PricePreference entity |
| `20240704094505` | PriceList metadata |
| `20241127114534` | BigNumber raw storage |
| `20241212190401` | PriceListRule JSON value |
| `20250408145122` | Performance indexes |
| `20251009110625` | Calculated price config cache |
| `20251112192723` | Tax-inclusive preference index |
| `20260429163502` | Rule operator enum expansion |
