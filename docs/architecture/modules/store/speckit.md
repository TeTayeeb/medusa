# SpecKit — Store Module

## Module Identity

| Attribute      | Value                                      |
|----------------|--------------------------------------------|
| Module Name    | `@medusajs/store`                          |
| Version        | 2.15.4                                     |
| Module Key     | `store`                                    |
| Constant       | `Modules.STORE`                            |
| Type           | Domain / Configuration                     |
| Database Tables| `store`, `store_currency`                  |
| Event Emitter  | Yes (`store.created`, `store.updated`, `store.deleted`) |
| Event Consumer | No                                         |

---

## Functional Specifications

### SPEC-STORE-001: Single Store Invariant
**Description**: At most one store record with `deleted_at IS NULL` may exist at any time.  
**Acceptance**: Two concurrent `POST /admin/stores` requests → one succeeds (201), one returns 400 with message `A store already exists`.

### SPEC-STORE-002: Supported Currencies
**Description**: The store MUST maintain a list of accepted ISO 4217 currency codes in `store_currency`. Operations MUST support add and remove.  
**Acceptance**: `addStoreCurrencies([{ currency_code: "eur" }])` → EUR appears in `supported_currencies` list.

### SPEC-STORE-003: Default Currency Validation
**Description**: `default_currency_code` MUST be present in the store's `supported_currencies` list. Setting a default currency not in the list MUST fail with `INVALID_DATA`.  
**Acceptance**: Store with `[usd, eur]` currencies; `update({ default_currency_code: "gbp" })` → 400.

### SPEC-STORE-004: Store Name
**Description**: `name` is a required field on store creation.  
**Acceptance**: `POST /admin/stores` without `name` → 400 validation error.

### SPEC-STORE-005: Metadata
**Description**: The `metadata` JSONB column MUST accept arbitrary key-value pairs and persist them without modification.  
**Acceptance**: `update({ metadata: { internal_id: "STORE_A" } })` → GET returns same metadata.

### SPEC-STORE-006: Admin API Full Access
**Description**: Authenticated admin users MUST be able to retrieve, create, and update store configuration via `/admin/stores`.  
**Acceptance**: All CRUD operations return correct status codes and updated entities.

### SPEC-STORE-007: Storefront Read Endpoint
**Description**: `GET /store/store` MUST return the store's public configuration (id, name, currencies) without sensitive metadata.  
**Acceptance**: Response contains `id`, `name`, `default_currency_code`, `supported_currencies`. Does NOT contain `metadata`.

### SPEC-STORE-008: Event Emission
**Description**: `store.created`, `store.updated`, and `store.deleted` events MUST be emitted on the event bus after successful operations.  
**Acceptance**: Event bus mock receives `store.updated` event after `updateStores()` called.

### SPEC-STORE-009: Cross-Module Links
**Description**: Default sales channel and region MUST be linked via Link Modules, not as FK columns on the `store` table.  
**Acceptance**: `GET /admin/stores` with `fields: ["default_sales_channel.*"]` returns sales channel data via Query helper.

---

## Non-Functional Specifications

### SPEC-STORE-NFR-001: Storefront Endpoint Cacheability
**Description**: `GET /store/store` response MUST be deterministic and suitable for CDN caching (no user-specific data).  
**Target**: `Cache-Control: public, max-age=60` header set on response.

### SPEC-STORE-NFR-002: Write Idempotency
**Description**: `updateStores()` called twice with the same data MUST produce the same result without errors.  
**Target**: Second call returns 200 with unchanged entity; no duplicate records.

---

## API Contract

### `GET /admin/stores`
**Auth**: Admin  
**Response**: `200` — `{ stores: Store[], count, limit, offset }`  
**Expandable fields**: `default_sales_channel`, `default_region`, `supported_currencies`

### `POST /admin/stores`
**Body**: `{ name: string, default_currency_code: string, supported_currencies?: [...] }`  
**Response**: `201` — `{ store: Store }` | `400` if store already exists

### `PUT /admin/stores/:id`
**Body**: Partial store update  
**Response**: `200` — `{ store: Store }` | `400` if currency validation fails

### `GET /store/store`
**Auth**: Public  
**Response**: `200` — `{ store: { id, name, default_currency_code, supported_currencies } }`

---

## Configuration Schema

No explicit module options required. Registered automatically as `Modules.STORE`.

---

## Test Checklist

- [ ] Single store invariant enforced (concurrent creation rejected)
- [ ] Currency not in supported list rejected as default
- [ ] `metadata` round-trips correctly
- [ ] `GET /store/store` excludes metadata
- [ ] `store.updated` event emitted after update
- [ ] Default sales channel resolved via Query helper
- [ ] Storefront endpoint has public Cache-Control header
- [ ] `addStoreCurrencies` upserts; no duplicates

---

## Dependencies & Interfaces

| Dependency       | Interface Used                         | Direction |
|------------------|----------------------------------------|-----------|
| Link Modules     | `StoreDefaultSalesChannelLink`         | Outbound  |
| Link Modules     | `StoreDefaultRegionLink`               | Outbound  |
| Event Bus        | Emit `store.*` events                  | Outbound  |
| Settings module  | Store-scoped settings bridge           | Inbound   |
