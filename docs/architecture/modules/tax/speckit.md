# Specification — Tax Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                                                   |
|-----------------|---------------------------------------------------------|
| Module ID       | `Modules.TAX`                                           |
| Package         | `@medusajs/tax`                                         |
| Medusa Version  | 2.15.4                                                  |
| Type            | Commerce Logic / Provider Abstraction                   |
| Database tables | `tax_region`, `tax_rate`, `tax_rate_rule`, `tax_provider` |
| API surface     | Admin only (CRUD for regions and rates)                 |

## 2. Functional Requirements

### FR-TAX-01: Create Tax Region
- **Given** a valid `country_code` (ISO 3166-1 alpha-2)
- **When** `POST /admin/tax-regions` is called
- **Then** a new TaxRegion is created; optionally with a `parent_id` for province-level regions

### FR-TAX-02: Create Province-Level Tax Region
- **Given** a parent TaxRegion (country level) and a `province_code` (ISO 3166-2)
- **When** `POST /admin/tax-regions` is called with `parent_id`
- **Then** a child TaxRegion is created; its rates supplement or override the parent's

### FR-TAX-03: Create Tax Rate
- **Given** an existing TaxRegion ID
- **When** `POST /admin/tax-rates` with `{ tax_region_id, rate, name, is_default }` is called
- **Then** a TaxRate is created and associated with the region

### FR-TAX-04: Default Rate Enforcement
- **Given** a TaxRegion
- **When** a new rate with `is_default: true` is created
- **Then** any previously default rate in the same region is updated to `is_default: false`

### FR-TAX-05: Create Tax Rate Rule
- **Given** a TaxRate ID and a `reference` (product / product_type / product_tag) + `reference_id`
- **When** `createTaxRateRules()` is called
- **Then** the rate is applied selectively to the referenced entity

### FR-TAX-06: Tax Line Calculation
- **Given** cart line items, shipping methods, and a calculation context (region, address)
- **When** `getTaxLines(items, shipping, context)` is called
- **Then** return `ItemTaxLineDTO[]` with `rate`, `name`, `code`, and computed `amount` per item/method

### FR-TAX-07: Rule-Based Rate Selection
- **Given** a line item with a `product_id`, `product_type_id`, and `product_tag_ids`
- **When** tax lines are calculated
- **Then** matching priority is: product → product_type → product_tag → default rate

### FR-TAX-08: External Provider Delegation
- **Given** a TaxRegion with a `provider_id` set
- **When** `getTaxLines()` is invoked
- **Then** calculation is delegated to the registered `ITaxProvider` plugin for that region

### FR-TAX-09: Combinable Rates
- **Given** parent (country) and child (province) TaxRegions each with rates
- **When** an address matches both regions
- **Then** rates marked `is_combinable: true` are summed; non-combinable rates replace parent rate

## 3. Non-Functional Requirements

| ID          | Requirement                       | Target                                              |
|-------------|-----------------------------------|-----------------------------------------------------|
| NFR-TAX-01  | Tax line calculation latency       | < 50ms (built-in calculation, no external call)    |
| NFR-TAX-02  | External provider call timeout     | Provider-defined; recommended < 2000ms             |
| NFR-TAX-03  | Rule lookup performance            | Indexed on `tax_rate_id`, `reference`, `reference_id` |
| NFR-TAX-04  | Concurrent checkout safety         | Read-only at calculation time; no write contention  |

## 4. Interface Specification

### POST `/admin/tax-regions`

| Attribute     | Value                                                                                       |
|---------------|---------------------------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                                             |
| Body          | `{ country_code: string, province_code?: string, parent_id?: string, provider_id?: string }` |
| Response 200  | `{ tax_region: TaxRegionDTO }`                                                              |

### POST `/admin/tax-rates`

| Attribute     | Value                                                                                                        |
|---------------|--------------------------------------------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                                                              |
| Body          | `{ tax_region_id: string, name: string, rate: number, code?: string, is_default?: boolean, is_combinable?: boolean }` |
| Response 200  | `{ tax_rate: TaxRateDTO }`                                                                                   |

## 5. Data Contracts

### TaxRegionDTO

```ts
type TaxRegionDTO = {
  id: string
  country_code: string
  province_code: string | null
  parent_id: string | null
  provider_id: string | null
  tax_rates: TaxRateDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### TaxRateDTO

```ts
type TaxRateDTO = {
  id: string
  name: string
  rate: number | null
  code: string | null
  is_default: boolean
  is_combinable: boolean
  tax_region_id: string
  rules: TaxRateRuleDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### TaxRateRuleDTO

```ts
type TaxRateRuleDTO = {
  id: string
  tax_rate_id: string
  reference: "product" | "product_type" | "product_tag"
  reference_id: string
  created_at: Date
  updated_at: Date
}
```

### ItemTaxLineDTO

```ts
type ItemTaxLineDTO = {
  line_item_id?: string
  shipping_method_id?: string
  rate: number
  name: string
  code: string | null
  provider_id: string
}
```

## 6. Edge Cases

| Case                                           | Expected Behaviour                                                |
|------------------------------------------------|-------------------------------------------------------------------|
| No TaxRegion for cart's country                | No tax lines returned (zero tax)                                  |
| Region has no default rate                     | No fallback; items with no matching rule get zero tax             |
| Province TaxRegion with `is_combinable: false` | Province rate replaces parent rate; parent rate ignored           |
| Product matches both product_type and product rule | Product-level rule takes precedence (highest specificity)    |
| External provider throws exception             | Error propagated; checkout step fails; cart not completed         |
| `rate: 0` tax rate                             | Valid; used for zero-rating exempt goods                          |
| Tax-inclusive pricing (`includes_tax: true`)   | Cart module adjusts line total; tax amount is extracted, not added |

## 7. Module Boundaries

| In Scope                                        | Out of Scope                                          |
|-------------------------------------------------|-------------------------------------------------------|
| Tax region/rate/rule management                 | VAT reporting or compliance filing                    |
| Rule-based rate selection per product/type/tag  | Currency conversion                                   |
| Provider abstraction for external engines       | Rate synchronisation with tax authority APIs          |
| Tax line computation for cart and shipping      | Withholding or payroll tax                            |

## 8. Acceptance Criteria Summary

- [ ] `POST /admin/tax-regions { country_code: "DE" }` creates a region
- [ ] Two rates in same region: `is_default` is unique (setting one clears the other)
- [ ] TaxRateRule with `reference: "product_type"` applies to all products of that type
- [ ] `getTaxLines()` for a German address returns 19% for standard products
- [ ] `getTaxLines()` for a food product with 7% rule returns 7%
- [ ] Province-level region (e.g., CA-QC) overrides/supplements country-level rates
- [ ] Region with `provider_id` delegates to registered `ITaxProvider`
- [ ] `DELETE /admin/tax-rates/:id` soft-deletes; rate no longer applies in calculations
