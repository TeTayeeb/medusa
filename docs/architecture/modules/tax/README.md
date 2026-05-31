# Tax Module

The Tax module (`@medusajs/tax`) provides a flexible, rule-based taxation system for Medusa. It supports hierarchical tax regions, multiple tax rates with fine-grained applicability rules, and a provider abstraction layer for integrating external tax calculation engines such as TaxJar or Avalara.

## Purpose

Tax calculation is one of the most complex aspects of commerce. The Tax module decouples tax logic from the cart and order processing pipeline: it defines _what_ rates apply (via TaxRegions and TaxRates) and _how_ they are calculated (via provider plugins), while the Cart module orchestrates when to call it.

## Key Features

- **Hierarchical Tax Regions** — Tax regions can be nested (e.g., country → province/state), with child regions inheriting and overriding parent settings.
- **Flexible Tax Rates** — Each tax region can carry multiple rates with a `rate` percentage and `is_default` flag.
- **Tax Rate Rules** — Rates can be targeted to specific products, product types, or product tags via `TaxRateRule`, enabling zero-rating or reduced rates for exempt goods.
- **Provider Abstraction** — Tax regions can delegate calculation to an external provider (e.g., TaxJar, Avalara) instead of using the built-in flat-rate logic.
- **Included vs Excluded Taxes** — Supports both gross (tax-included) and net (tax-excluded) pricing models.
- **Line Item and Shipping Tax** — Tax lines are calculated for both cart line items and shipping methods.

## Entities

| Entity          | Key Fields                                                                                    |
|-----------------|-----------------------------------------------------------------------------------------------|
| `TaxRegion`     | `id`, `country_code`, `province_code`, `parent_id`, `provider_id`                           |
| `TaxRate`       | `id`, `name`, `rate`, `code`, `is_default`, `is_combinable`, `tax_region_id`                |
| `TaxRateRule`   | `id`, `tax_rate_id`, `reference_id`, `reference` (product/product_type/product_tag)         |
| `TaxProvider`   | `id`, `is_enabled` (reference entity for registered provider plugins)                        |

## Admin API

| Method | Endpoint                       | Description                        |
|--------|--------------------------------|------------------------------------|
| GET    | `/admin/tax-regions`           | List tax regions                   |
| POST   | `/admin/tax-regions`           | Create a tax region                |
| GET    | `/admin/tax-regions/:id`       | Retrieve a tax region              |
| DELETE | `/admin/tax-regions/:id`       | Delete a tax region                |
| GET    | `/admin/tax-rates`             | List tax rates                     |
| POST   | `/admin/tax-rates`             | Create a tax rate                  |
| GET    | `/admin/tax-rates/:id`         | Retrieve a tax rate                |
| POST   | `/admin/tax-rates/:id`         | Update a tax rate                  |
| DELETE | `/admin/tax-rates/:id`         | Delete a tax rate                  |

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.TAX
```

## Service Usage

```ts
const taxService = container.resolve(Modules.TAX)

// Create a tax region for Germany
const deRegion = await taxService.createTaxRegions({
  country_code: "DE",
})

// Create a standard 19% VAT rate
const vatRate = await taxService.createTaxRates({
  tax_region_id: deRegion.id,
  name: "Standard VAT",
  rate: 19,
  code: "DE_VAT_STANDARD",
  is_default: true,
})

// Create a reduced rate rule for food products
await taxService.createTaxRateRules({
  tax_rate_id: vatRate.id,
  reference: "product_type",
  reference_id: "ptyp_food",
})
```

## Module Links

| Link                         | Description                                              |
|------------------------------|----------------------------------------------------------|
| `region ↔ tax-region`        | Connects a Medusa Region to its corresponding TaxRegion  |

## Related Modules

- **Region Module** — Regions link to tax regions to determine which tax rules apply per market.
- **Cart Module** — Calls `getTaxLines()` during checkout to compute tax lines for line items and shipping.
- **Product Module** — Product types and tags are referenced in TaxRateRule for targeted rate application.

## Version

Medusa v2.15.4
