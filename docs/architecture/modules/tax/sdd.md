# Software Design Document — Tax Module

## 1. Overview

The Tax module implements Medusa's taxation system. It manages the hierarchical taxonomy of tax regions, the rates applicable within each region, the rules that target specific products or product types to non-default rates, and a provider abstraction that allows delegating calculation to external services. Tax line computation is the primary runtime output consumed by the Cart module during checkout.

## 2. Goals and Non-Goals

**Goals:**
- Maintain a hierarchy of tax regions (country → province/state).
- Support multiple named tax rates per region with default and override logic.
- Enable product/product_type/product_tag-level rate targeting via rules.
- Abstract the calculation engine behind a `ITaxProvider` interface.
- Compute tax lines for cart line items and shipping methods.

**Non-Goals:**
- VAT reporting or compliance filing.
- Currency conversion.
- Rate synchronisation with external tax authorities (provider responsibility).
- Withholding tax or employer-side tax calculations.

## 3. Data Model

### 3.1 TaxRegion Entity

```ts
@Entity()
export class TaxRegion extends BaseEntity {
  @PrimaryKey()
  id: string  // ULID

  @Property()
  country_code: string   // ISO 3166-1 alpha-2, e.g. "DE"

  @Property({ nullable: true })
  province_code?: string  // ISO 3166-2 province, e.g. "CA-BC"

  @ManyToOne(() => TaxRegion, { nullable: true })
  parent?: TaxRegion

  @Property({ nullable: true })
  parent_id?: string

  @Property({ nullable: true })
  provider_id?: string   // References TaxProvider.id

  @OneToMany(() => TaxRate, (r) => r.tax_region)
  tax_rates = new Collection<TaxRate>(this)
}
```

### 3.2 TaxRate Entity

```ts
@Entity()
export class TaxRate extends BaseEntity {
  @PrimaryKey()
  id: string

  @Property()
  name: string

  @Property({ nullable: true })
  rate?: number   // Percentage, e.g. 19

  @Property({ nullable: true })
  code?: string   // Human identifier, e.g. "DE_VAT_19"

  @Property({ default: false })
  is_default: boolean

  @Property({ default: false })
  is_combinable: boolean

  @ManyToOne(() => TaxRegion)
  tax_region: TaxRegion

  @Property()
  tax_region_id: string

  @OneToMany(() => TaxRateRule, (r) => r.tax_rate)
  rules = new Collection<TaxRateRule>(this)
}
```

### 3.3 TaxRateRule Entity

```ts
@Entity()
export class TaxRateRule extends BaseEntity {
  @PrimaryKey()
  id: string

  @ManyToOne(() => TaxRate)
  tax_rate: TaxRate

  @Property()
  tax_rate_id: string

  @Property()
  reference: string      // "product" | "product_type" | "product_tag"

  @Property()
  reference_id: string   // ID of the referenced entity
}
```

### 3.4 TaxProvider Entity

```ts
@Entity()
export class TaxProvider {
  @PrimaryKey()
  id: string   // e.g. "taxjar" or "system"

  @Property({ default: true })
  is_enabled: boolean
}
```

## 4. Service Interface

```ts
interface ITaxModuleService {
  createTaxRegions(data: CreateTaxRegionDTO[], sharedContext?: Context): Promise<TaxRegionDTO[]>
  updateTaxRegions(data: UpdateTaxRegionDTO[], sharedContext?: Context): Promise<TaxRegionDTO[]>
  deleteTaxRegions(ids: string[], sharedContext?: Context): Promise<void>

  createTaxRates(data: CreateTaxRateDTO[], sharedContext?: Context): Promise<TaxRateDTO[]>
  updateTaxRates(data: UpdateTaxRateDTO[], sharedContext?: Context): Promise<TaxRateDTO[]>
  deleteTaxRates(ids: string[], sharedContext?: Context): Promise<void>

  createTaxRateRules(data: CreateTaxRateRuleDTO[], sharedContext?: Context): Promise<TaxRateRuleDTO[]>
  deleteTaxRateRules(ids: string[], sharedContext?: Context): Promise<void>

  getTaxLines(
    items: TaxableItemDTO[],
    shippingMethods: TaxableShippingDTO[],
    context: TaxCalculationContext,
    sharedContext?: Context
  ): Promise<ItemTaxLineDTO[]>
}
```

## 5. Tax Calculation Flow

```
Cart checkout trigger
      │
      ▼
getTaxLines(lineItems, shippingMethods, { region, customer, address })
      │
      ├─ Resolve TaxRegion for region.country_code (+ province if present)
      ├─ Walk region hierarchy: province > country > default
      ├─ Match item against TaxRateRules (product_type → product_tag → product)
      ├─ If provider_id set on TaxRegion → delegate to ITaxProvider.getTaxLines()
      └─ Else compute using flat rate: amount × (rate / 100)
      │
      ▼
ItemTaxLineDTO[] returned to Cart module
```

## 6. Provider Interface

```ts
interface ITaxProvider {
  identifier: string
  getTaxLines(
    itemLines: TaxableItemDTO[],
    shippingLines: TaxableShippingDTO[],
    context: TaxCalculationContext
  ): Promise<ProviderTaxLineDTO[]>
}
```

Custom providers implement this interface and register under `Modules.TAX` provider configuration.

## 7. Hierarchy Resolution

When looking up the applicable TaxRegion for a cart:
1. Attempt to match `country_code` + `province_code` → province-level TaxRegion.
2. Fall back to `country_code`-only TaxRegion.
3. The child (province) region's rates **supplement** parent rates unless `is_combinable = false`.

## 8. Error Handling

| Condition                             | Error Type                        |
|---------------------------------------|-----------------------------------|
| TaxRegion not found                   | `MedusaError.Types.NOT_FOUND`     |
| TaxRate not found                     | `MedusaError.Types.NOT_FOUND`     |
| Invalid `reference` in TaxRateRule    | `MedusaError.Types.INVALID_DATA`  |
| Duplicate default rate in region      | `MedusaError.Types.INVALID_DATA`  |
| Provider plugin not registered        | `MedusaError.Types.NOT_FOUND`     |
