# Software Design Document — Store Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa Store module (v2.15.4). It covers the entity model, service API, cross-module link resolution, API routes, and the single-record invariant that governs store configuration.

## 2. Architecture Overview

The Store module is a thin configuration module with two primary entities (`Store` and `StoreCurrency`) and a service that manages them. Its design is intentionally simple: most complexity lives in the cross-module links managed by the Link Modules system.

```
Admin Dashboard / Storefront
  ↓
GET /admin/stores | PUT /admin/stores/:id | GET /store/store
  ↓
StoreModuleService
  ├─ listStores() / retrieveStore()
  ├─ createStore() / updateStore()
  └─ addStoreCurrencies() / removeStoreCurrencies()
      ↓
Store Entity ← StoreCurrency Entity (1-to-many)
      ↓
Query Helper → StoreDefaultSalesChannelLink
             → StoreDefaultRegionLink
```

## 3. Data Model

### 3.1 Store

| Field                    | Type          | Notes                                     |
|--------------------------|---------------|-------------------------------------------|
| `id`                     | string        | `store_*` prefix                          |
| `name`                   | string        | Display name                              |
| `default_currency_code`  | string        | ISO 4217; must exist in `supported_currencies` |
| `supported_currencies`   | StoreCurrency[] | Eager loaded                            |
| `metadata`               | JSONB         | Arbitrary merchant metadata               |
| `created_at`             | timestamp     | —                                         |
| `updated_at`             | timestamp     | —                                         |
| `deleted_at`             | timestamp     | Soft delete                               |

### 3.2 StoreCurrency

| Field           | Type    | Notes                                   |
|-----------------|---------|-----------------------------------------|
| `id`            | string  | `stcur_*` prefix                        |
| `store_id`      | string  | FK to `store.id`                        |
| `currency_code` | string  | ISO 4217                                |
| `is_default`    | boolean | True for the store's default currency   |
| `created_at`    | timestamp | —                                     |
| `deleted_at`    | timestamp | Soft delete                           |

## 4. Single-Record Invariant

A Medusa deployment has exactly one active store record. This is enforced by:

1. A unique partial index on `store` where `deleted_at IS NULL`.
2. `StoreModuleService.createStore()` checking for an existing record before inserting.

Multi-store architectures are not supported natively; the recommended pattern is separate Medusa instances.

## 5. Service API

```typescript
interface IStoreModuleService {
  retrieveStore(id: string, config?: FindConfig<StoreDTO>): Promise<StoreDTO>
  listStores(filters?: FilterableStoreProps, config?: FindConfig<StoreDTO>): Promise<StoreDTO[]>
  createStores(data: CreateStoreDTO[]): Promise<StoreDTO[]>
  updateStores(data: UpdateStoreDTO[]): Promise<StoreDTO[]>
  deleteStores(ids: string[]): Promise<void>
  addStoreCurrencies(data: AddStoreCurrenciesDTO[]): Promise<StoreCurrencyDTO[]>
  removeStoreCurrencies(currencyIds: string[]): Promise<void>
  upsertStores(data: UpsertStoreDTO[]): Promise<StoreDTO[]>
}
```

## 6. Default Currency Validation

When updating `default_currency_code`, the service verifies that the new code exists in the store's `supported_currencies` list:

```typescript
if (!store.supported_currencies.some(c => c.currency_code === dto.default_currency_code)) {
  throw new MedusaError(
    MedusaError.Types.INVALID_DATA,
    `Currency ${dto.default_currency_code} is not in the store's supported currencies`
  )
}
```

## 7. Cross-Module Links

The store does not hold direct FKs to `SalesChannel` or `Region`. These relationships are managed by Link Modules:

| Link                            | Resolution Query                             |
|---------------------------------|----------------------------------------------|
| `StoreDefaultSalesChannelLink`  | `fields: ["default_sales_channel.*"]`       |
| `StoreDefaultRegionLink`        | `fields: ["default_region.*"]`              |

Updating these links involves calling the link service, not the store service directly:

```typescript
await remoteLink.create({
  [Modules.STORE]: { store_id: storeId },
  [Modules.SALES_CHANNEL]: { sales_channel_id: scId },
})
```

## 8. API Route Design

| Route              | Handler Logic                                               |
|--------------------|-------------------------------------------------------------|
| `GET /admin/stores` | `listStores()` with full field expansion                   |
| `POST /admin/stores` | `createStores()` → validates single-record invariant      |
| `PUT /admin/stores/:id` | `updateStores()` → currency validation → link updates |
| `GET /store/store`  | `listStores()` with public-safe field projection           |

The storefront `GET /store/store` strips sensitive metadata and exposes only: `id`, `name`, `default_currency_code`, `supported_currencies`.

## 9. Event Emission

The store service emits the following events:

| Event                 | Payload              |
|-----------------------|----------------------|
| `store.created`       | `{ id }`             |
| `store.updated`       | `{ id }`             |
| `store.deleted`       | `{ id }`             |

## 10. Error Handling

| Scenario                       | Error Type        | Message                              |
|--------------------------------|-------------------|--------------------------------------|
| Store not found                | NOT_FOUND         | `Store with id X was not found`     |
| Duplicate store creation       | INVALID_DATA      | `A store already exists`            |
| Invalid default currency       | INVALID_DATA      | `Currency X not in supported list`  |
