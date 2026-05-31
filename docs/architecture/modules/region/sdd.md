# Software Design Document — Region Module

## 1. Overview

The Region module manages the geographical and commercial configuration boundaries that Medusa uses to serve different markets. A region is the primary unit of market configuration: it groups countries, associates a currency, sets tax defaults, and is referenced by carts, orders, and the tax system. The module implements full CRUD operations exposed over both the Admin and Store APIs.

## 2. Goals and Non-Goals

**Goals:**
- Group countries into regions for shared commerce configuration.
- Associate a currency code with each region.
- Store default tax rate and automatic tax calculation preferences per region.
- Provide a stable reference for the Cart and Order modules.
- Expose a Store API so storefronts can present available regions to customers.

**Non-Goals:**
- Pricing logic (delegated to Pricing module).
- Tax rule details (delegated to Tax module; Region only stores top-level defaults).
- Payment and fulfillment provider management (managed via module links, not Region module internals).

## 3. Data Model

### 3.1 Region Entity

```ts
@Entity()
export class Region extends BaseEntity {
  @PrimaryKey()
  id: string  // ULID

  @Property()
  name: string

  @Property()
  currency_code: string  // References Currency.code

  @Property({ default: true })
  automatic_taxes: boolean

  @Property({ nullable: true })
  tax_rate?: number

  @Property({ nullable: true })
  tax_code?: string

  @OneToMany(() => RegionCountry, (c) => c.region)
  countries = new Collection<RegionCountry>(this)

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.2 RegionCountry Entity

```ts
@Entity()
export class RegionCountry extends BaseEntity {
  @PrimaryKey()
  id: string

  @Property({ unique: true })
  iso_2: string   // Two-letter ISO 3166-1 code

  @Property()
  iso_3: string   // Three-letter ISO 3166-1 code

  @Property()
  num_code: string

  @Property()
  name: string

  @Property()
  display_name: string

  @ManyToOne(() => Region)
  region: Region

  @Property()
  region_id: string
}
```

### 3.3 Database Tables

| Table            | Primary Key | Notable Indexes                   |
|------------------|-------------|-----------------------------------|
| `region`         | `id` (ULID) | `deleted_at`                      |
| `region_country` | `id` (ULID) | `iso_2` (unique), `region_id` FK  |

## 4. Service Interface

```ts
interface IRegionModuleService {
  createRegions(data: CreateRegionDTO[], sharedContext?: Context): Promise<RegionDTO[]>
  updateRegions(data: UpdateRegionDTO[], sharedContext?: Context): Promise<RegionDTO[]>
  deleteRegions(ids: string[], sharedContext?: Context): Promise<void>
  retrieveRegion(id: string, config?: FindConfig<RegionDTO>, sharedContext?: Context): Promise<RegionDTO>
  listRegions(filters?: FilterableRegionProps, config?: FindConfig<RegionDTO>, sharedContext?: Context): Promise<RegionDTO[]>
  listAndCountRegions(filters?: FilterableRegionProps, config?: FindConfig<RegionDTO>, sharedContext?: Context): Promise<[RegionDTO[], number]>

  upsertRegionCountries(data: UpsertRegionCountryDTO[], sharedContext?: Context): Promise<RegionCountryDTO[]>
  deleteRegionCountries(ids: string[], sharedContext?: Context): Promise<void>
}
```

## 5. Module Architecture

```
@medusajs/region
├── src/
│   ├── models/
│   │   ├── region.ts
│   │   └── region-country.ts
│   ├── services/
│   │   └── region-module-service.ts
│   ├── migrations/
│   └── index.ts
```

`RegionModuleService` extends `MedusaService<{ Region: { dto: RegionDTO }, RegionCountry: { dto: RegionCountryDTO } }>`. The service uses `@InjectManager()` for public methods and `@InjectTransactionManager()` for protected internal operations to ensure atomic writes.

## 6. Workflows

Region CRUD operations are wrapped in core-flow workflows:

| Workflow                     | Steps                                         |
|------------------------------|-----------------------------------------------|
| `createRegionsWorkflow`      | `createRegionsStep`                           |
| `updateRegionsWorkflow`      | `updateRegionsStep`                           |
| `deleteRegionsWorkflow`      | `deleteRegionsStep`                           |

Each workflow emits a domain hook (`regionsCreated`, `regionsUpdated`, `regionsDeleted`) to allow subscribers to react (e.g., syncing tax regions).

## 7. Validation Rules

| Field           | Rule                                                             |
|-----------------|------------------------------------------------------------------|
| `currency_code` | Must match an existing Currency record (validated via remote query) |
| `iso_2`         | Must be a valid 2-letter ISO 3166-1 country code                 |
| `tax_rate`      | Must be between 0 and 100 if provided                            |
| Country uniqueness | A country can only belong to one region at a time             |

## 8. Error Handling

| Condition                            | Error Type                        |
|--------------------------------------|-----------------------------------|
| Region not found                     | `MedusaError.Types.NOT_FOUND`     |
| Invalid currency_code                | `MedusaError.Types.INVALID_DATA`  |
| Duplicate country assignment         | `MedusaError.Types.INVALID_DATA`  |
| Delete region with active carts      | Allowed (carts retain snapshot)   |

## 9. Integration with Cart Module

When a cart is created, it is assigned a `region_id`. The Cart module resolves the full Region to determine:
- Which currency to display prices in
- Whether to calculate taxes automatically
- Which payment/fulfillment providers are available

Region data is snapshotted into the order at completion to preserve historical accuracy.

## 10. Store API Design

The Store Region API is intentionally read-only. Customers may list available regions (to select their delivery market) and retrieve a specific region to confirm currency and locale details. All write operations are Admin-only.
