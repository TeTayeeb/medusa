# Region Module

The Region module (`@medusajs/region`) manages geographical groupings used to configure commerce behaviour per market. A region bundles currency, tax settings, and payment/fulfillment provider availability for a set of countries. It is a foundational module consumed by the Cart, Order, and Tax modules.

## Purpose

Medusa regions allow a single store to serve multiple markets with distinct pricing currencies, tax policies, and payment methods. A customer's active region determines which currency prices are shown, which tax rules apply, and which checkout providers are available.

## Key Features

- **Country Grouping** — Multiple countries can belong to one region, enabling shared configuration across similar markets.
- **Currency Association** — Each region is linked to one currency code (via the Currency module).
- **Tax Configuration** — Regions carry a default tax rate, tax code, and a flag for automatic vs. manual tax calculation.
- **Provider Association** — Payment and fulfillment providers are scoped per region through module links.
- **Soft Delete** — Regions can be deactivated without removing historical order references.

## Entities

| Entity          | Key Fields                                                                          |
|-----------------|-------------------------------------------------------------------------------------|
| `Region`        | `id`, `name`, `currency_code`, `automatic_taxes`, `tax_rate`, `tax_code`           |
| `RegionCountry` | `id`, `iso_2`, `iso_3`, `num_code`, `name`, `display_name`, `region_id`            |

## Admin API

| Method | Endpoint             | Description                          |
|--------|----------------------|--------------------------------------|
| GET    | `/admin/regions`     | List regions                         |
| POST   | `/admin/regions`     | Create a region                      |
| GET    | `/admin/regions/:id` | Retrieve a region                    |
| POST   | `/admin/regions/:id` | Update a region                      |
| DELETE | `/admin/regions/:id` | Delete a region                      |

## Store API

| Method | Endpoint            | Description                                      |
|--------|---------------------|--------------------------------------------------|
| GET    | `/store/regions`    | List regions (used to select customer region)    |
| GET    | `/store/regions/:id`| Retrieve a region                                |

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.REGION
```

## Service Usage

```ts
const regionService = container.resolve(Modules.REGION)

// Create a region
const region = await regionService.createRegions({
  name: "Europe",
  currency_code: "EUR",
  automatic_taxes: true,
})

// Add countries to a region
await regionService.upsertRegionCountries([
  { region_id: region.id, iso_2: "DE" },
  { region_id: region.id, iso_2: "FR" },
])

// List regions with countries
const regions = await regionService.listRegions({}, {
  relations: ["countries"]
})
```

## Data Shape

```ts
type RegionDTO = {
  id: string
  name: string
  currency_code: string
  automatic_taxes: boolean
  tax_rate: number | null
  tax_code: string | null
  countries: RegionCountryDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}

type RegionCountryDTO = {
  id: string
  iso_2: string
  iso_3: string
  num_code: string
  name: string
  display_name: string
  region_id: string
}
```

## Related Modules

- **Currency Module** — Validates and resolves `currency_code`.
- **Tax Module** — TaxRegion records are linked to Region for tax rule application.
- **Cart Module** — Cart carries a `region_id` that drives currency and tax behaviour.
- **Payment Module** — Payment providers are associated to regions via module links.

## Version

Medusa v2.15.4
