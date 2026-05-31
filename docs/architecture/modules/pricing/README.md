# Pricing Module

**Package:** `@medusajs/pricing`  
**Module key:** `Modules.PRICING`  
**Version:** Medusa v2.15.4

---

## Overview

The Pricing module is the central engine for all money-amount computation in Medusa. It models prices not as simple scalar values attached to products, but as **PriceSets** — collections of prices that are evaluated at query time against a **pricing context** (currency, region, customer group, quantity, etc.) to return the single best matching price.

This design enables sophisticated scenarios such as: regional pricing, volume discounts, B2B tiered pricing, time-bounded sale price lists, and currency-specific tax-inclusion preferences — all without duplicating product data.

---

## Key Entities

| Entity | DB table | ID prefix | Description |
|---|---|---|---|
| `PriceSet` | `price_set` | `pset_` | Container grouping all prices for a given product variant or shippable item |
| `Price` | `price` | `price_` | A concrete money amount with currency code, optional quantity bounds, and rule conditions |
| `PriceRule` | `price_rule` | `prule_` | A contextual condition on a `Price` (e.g. `region_id = reg_01`) |
| `PriceList` | `price_list` | `plist_` | A named override list (SALE or OVERRIDE type) with optional date range |
| `PriceListRule` | `price_list_rule` | `prule_` | A condition that must match for a `PriceList` to apply |
| `PricePreference` | `price_preference` | `prpref_` | Per-attribute/value flag for tax-inclusive pricing |

### PriceSet Fields
`id`, `prices` (hasMany → Price).

### Price Fields
`id`, `title`, `currency_code`, `amount` (BigNumber), `min_quantity` (nullable), `max_quantity` (nullable), `rules_count`, `price_set` (belongsTo), `price_rules` (hasMany), `price_list` (belongsTo, nullable).

### PriceRule Fields
`id`, `attribute`, `value`, `operator` (EQ | GT | GTE | LT | LTE | IN), `priority`, `price` (belongsTo).

### PriceList Fields
`id`, `title`, `description`, `status` (DRAFT | ACTIVE), `type` (SALE | OVERRIDE), `starts_at`, `ends_at`, `rules_count`, `prices` (hasMany), `price_list_rules` (hasMany), `metadata`.

### PricePreference Fields
`id`, `attribute`, `value`, `is_tax_inclusive` (boolean, default false). Unique on `(attribute, value)`.

---

## Key Service Methods

```ts
// PriceSet management
createPriceSets(data: CreatePriceSetDTO | CreatePriceSetDTO[]): Promise<PriceSetDTO | PriceSetDTO[]>
upsertPriceSets(data: UpsertPriceSetDTO[]): Promise<PriceSetDTO[]>
addPrices(data: AddPricesDTO): Promise<PriceSetDTO>
listPriceSets(filters?, config?): Promise<PriceSetDTO[]>
deletePriceSets(ids: string[]): Promise<void>

// Price calculation (core computation)
calculatePrices(
  filters: PricingFilters,        // { id: string[] } — PriceSet IDs
  context: PricingContext          // { currency_code, region_id, customer_group_id, quantity, ... }
): Promise<CalculatedPriceSet[]>

// PriceList management
createPriceLists(data): Promise<PriceListDTO | PriceListDTO[]>
updatePriceLists(id | selector, data): Promise<PriceListDTO | PriceListDTO[]>
deletePriceLists(ids: string[]): Promise<void>

// PricePreference management
upsertPricePreferences(data: UpsertPricePreferenceDTO[]): Promise<PricePreferenceDTO[]>
listPricePreferences(filters?, config?): Promise<PricePreferenceDTO[]>
```

---

## Price Calculation Algorithm

`calculatePrices` applies the following precedence logic:

1. Filter candidate `Price` rows that belong to the requested `PriceSet` IDs.
2. For each candidate, evaluate all attached `PriceRule` entries against the provided context. A price matches only when **all** of its rules pass.
3. Among matching prices, prefer those with the highest `rules_count` (most specific match wins).
4. Among equal `rules_count`, apply `PriceList` override logic (OVERRIDE type takes precedence over SALE, which takes precedence over base prices).
5. Check `PriceList.starts_at` / `ends_at` — inactive lists are excluded.
6. Apply `PricePreference.is_tax_inclusive` for the resolved `currency_code`.

---

## API Endpoints

### Admin API
| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/price-lists` | List price lists |
| `POST` | `/admin/price-lists` | Create a price list |
| `GET` | `/admin/price-lists/:id` | Retrieve a price list |
| `POST` | `/admin/price-lists/:id` | Update a price list |
| `DELETE` | `/admin/price-lists/:id` | Delete a price list |
| `POST` | `/admin/price-lists/:id/prices/batch` | Batch-add prices to a list |
| `GET` | `/admin/price-preferences` | List price preferences |
| `POST` | `/admin/price-preferences` | Upsert price preferences |
| `DELETE` | `/admin/price-preferences/:id` | Delete a preference |

> **Note:** Prices for product variants are returned inline in `/admin/products` and `/store/products` responses via the remote-query expansion.

---

## Module Links

- **Product module** → `ProductVariant` is linked to a `PriceSet` via a Module Link (no foreign key in either module's DB)
- **Shipping module** → `ShippingOption` cost linked to a `PriceSet`
- **Customer module** → `CustomerGroup.id` used as `PriceRule.attribute = "customer_group_id"`
- **Region module** → `Region.id` used as `PriceRule.attribute = "region_id"`

---

## Events Emitted

| Event | Payload | Trigger |
|---|---|---|
| `price_set.created` | `{ id }` | After `createPriceSets` |
| `price_set.deleted` | `{ id }` | After `deletePriceSets` |
| `price_list.created` | `{ id }` | After `createPriceLists` |
| `price_list.updated` | `{ id }` | After `updatePriceLists` |
| `price_list.deleted` | `{ id }` | After `deletePriceLists` |

---

## Related Modules

- **Product** — variants linked to PriceSets via Module Links
- **Customer** — customer groups used as rule attributes
- **Region** — region IDs used as rule attributes
- **Cart** — checkout reads calculated prices for line items
- **Order** — order prices derived from pricing context at time of placement
