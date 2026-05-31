# Software Design Document — Currency Module

## 1. Overview

The Currency module provides a canonical reference store for ISO 4217 currency definitions. It is a passive, infrastructure-level module: it does not initiate business logic, does not emit domain events, and exposes a minimal read-focused service interface. All other modules that deal with monetary amounts depend on this module for currency metadata.

## 2. Goals and Non-Goals

**Goals:**
- Provide consistent, centralized currency metadata to all Medusa modules.
- Expose a queryable service interface supporting filtering, sorting, and pagination.
- Maintain decimal precision metadata to support correct monetary arithmetic across the platform.
- Support soft-delete to allow future deprecation of currencies without breaking references.

**Non-Goals:**
- Exchange rate management or currency conversion.
- Locale-based formatting or number display logic.
- Write operations exposed via public API.
- Historical currency tracking or versioning.

## 3. Data Model

### 3.1 Currency Entity

```ts
@Entity()
export class Currency extends BaseEntity {
  @PrimaryKey()
  code: string  // ISO 4217 three-letter code (natural PK, not UUID)

  @Property()
  name: string

  @Property()
  symbol: string

  @Property()
  symbol_native: string

  @Property()
  decimal_digits: number

  @Property({ nullable: true })
  raw_decimal_digits?: Record<string, unknown>

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.2 Database Table

- **Table name**: `currency`
- **Primary key**: `code` (string, ISO 4217 three-letter code) — a natural key used directly by referencing modules
- **No foreign keys**: currencies are leaf reference entities
- **Soft delete**: `deleted_at` column; excluded from default queries when set

### 3.3 DTO

```ts
type CurrencyDTO = {
  code: string
  name: string
  symbol: string
  symbol_native: string
  decimal_digits: number
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

## 4. Service Interface

```ts
interface ICurrencyModuleService {
  listCurrencies(
    filters?: FilterableCurrencyProps,
    config?: FindConfig<CurrencyDTO>,
    sharedContext?: Context
  ): Promise<CurrencyDTO[]>

  listAndCountCurrencies(
    filters?: FilterableCurrencyProps,
    config?: FindConfig<CurrencyDTO>,
    sharedContext?: Context
  ): Promise<[CurrencyDTO[], number]>

  retrieveCurrency(
    code: string,
    config?: FindConfig<CurrencyDTO>,
    sharedContext?: Context
  ): Promise<CurrencyDTO>
}
```

### 4.1 FilterableCurrencyProps

```ts
type FilterableCurrencyProps = {
  code?: string | string[]
  name?: string
}
```

All methods accept an optional `sharedContext: Context` for transaction propagation across module boundaries.

## 5. Module Architecture

```
@medusajs/currency
├── src/
│   ├── models/
│   │   └── currency.ts                  # MikroORM entity
│   ├── services/
│   │   └── currency-module-service.ts   # Extends MedusaService<T>
│   ├── migrations/
│   │   └── Migration*.ts                # Database schema migrations
│   ├── scripts/
│   │   └── seed.ts                      # ISO 4217 seed script
│   └── index.ts                         # Module definition and exports
```

The service class extends `MedusaService<{ Currency: { dto: CurrencyDTO } }>` from `@medusajs/framework/utils`, which auto-generates standard CRUD operations (list, listAndCount, retrieve) via the repository pattern.

## 6. Seeding Strategy

Currency data is seeded via a module seed script that runs at application startup. The script:
1. Loads the static ISO 4217 currency list from a bundled JSON file.
2. Performs an `upsertCurrencies()` call, making the operation idempotent.
3. Does not delete currencies that have been soft-deleted (preserves manual deprecations).

New Medusa installations receive all ~170 active ISO 4217 currencies. Custom deployments can extend or replace the seed data.

## 7. Read Path

```
GET /admin/currencies
      │
      ▼
Admin Route Handler (packages/medusa/src/api/admin/currencies/route.ts)
      │  resolves ICurrencyModuleService from container
      ▼
CurrencyModuleService.listCurrencies(filters, config)
      │
      ▼
MikroORM Repository → SELECT * FROM currency WHERE ...
      │
      ▼
CurrencyDTO[] → JSON response
```

## 8. Error Handling

| Condition                        | Error Type                          | HTTP Status |
|----------------------------------|-------------------------------------|-------------|
| Currency code not found          | `MedusaError.Types.NOT_FOUND`       | 404         |
| Invalid filter value type        | Validation middleware rejection     | 400         |
| Write operation attempted via API| Route not defined (no POST/DELETE)  | 404/405     |

## 9. Caching Considerations

Since currency data changes rarely, it is an ideal candidate for:
- **HTTP cache headers**: `Cache-Control: public, max-age=86400` on the Admin API list endpoint.
- **Application-level caching**: In-memory cache keyed by `code` for `retrieveCurrency()` hot paths.
- The full dataset (~170 records) fits comfortably in a single cached response.

## 10. Performance Characteristics

- **List all**: ~1ms query, single table scan, no joins.
- **Retrieve by code**: Primary key lookup, O(1) database operation.
- **Payload size**: Full list ~25KB uncompressed, ~5KB gzipped.

## 11. Integration Pattern

Other modules do **not** embed currency data. They store only the `currency_code` string (e.g., `money_amount.currency_code = "USD"`) and resolve the full `CurrencyDTO` on demand through either:
- Direct container resolution: `container.resolve(Modules.CURRENCY).retrieveCurrency(code)`
- Remote query: `query.graph({ entity: "currency", filters: { code } })`

This avoids data duplication and ensures consistent metadata.
