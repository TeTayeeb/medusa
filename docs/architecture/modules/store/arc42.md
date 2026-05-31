# arc42 Architecture Document — Store Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Store module must manage the single top-level configuration record for a Medusa installation, enforce the single-record invariant, maintain a supported currency list, and expose both admin and storefront APIs for store configuration.

### 1.2 Quality Goals

| Priority | Quality Goal   | Scenario                                                         |
|----------|----------------|------------------------------------------------------------------|
| 1        | Integrity      | Only one store record exists; duplicate creation is rejected     |
| 2        | Correctness    | `default_currency_code` must always be in `supported_currencies` |
| 3        | Discoverability| Storefront can bootstrap currency/locale from `GET /store/store` |
| 4        | Extensibility  | Cross-module defaults (region, sales channel) added via links    |

### 1.3 Stakeholders

| Role              | Expectation                                              |
|-------------------|----------------------------------------------------------|
| Merchant          | Configure store name, currencies, and defaults via admin |
| Storefront dev    | Reliable endpoint to fetch store locale/currency config  |
| Platform operator | Single config record as root anchor for deployments      |

---

## 2. Architecture Constraints

- Only one `store` record may exist with `deleted_at IS NULL`.
- Default region and sales channel are not stored as FK columns; they are managed by Link Modules.
- The store module must not import from the Sales Channel or Region modules.

---

## 3. System Scope and Context

```
┌──────────────────────┐     ┌──────────────────────┐
│   Admin Dashboard    │     │     Storefront        │
│  PUT /admin/stores   │     │  GET /store/store     │
└──────────┬───────────┘     └──────────┬────────────┘
           │                             │
           └──────────────┬──────────────┘
                          │
          ┌───────────────▼───────────────┐
          │       StoreModuleService      │
          │  createStores / updateStores  │
          │  addStoreCurrencies           │
          └───────────────┬───────────────┘
                          │
          ┌───────────────▼───────────────┐
          │   store + store_currency (DB) │
          └───────────────────────────────┘
                          │ Link Modules
          ┌───────────────▼───────────────┐
          │  SalesChannel / Region modules│
          └───────────────────────────────┘
```

---

## 4. Solution Strategy

- Store is a **configuration module**: mostly read-heavy, write-rare.
- **Single-record pattern**: a unique partial index enforces the invariant at the DB level.
- **Link Modules** for all cross-domain relationships; no direct FK outside the store schema.
- **Two API surfaces**: admin (full write access) and store (public read, restricted fields).

---

## 5. Building Block View

### Level 1

```
StoreModule
  ├── StoreModuleService        (MedusaService<{Store, StoreCurrency}>)
  ├── Store                     (entity)
  ├── StoreCurrency             (entity)
  └── Route handlers
       ├── GET/POST/PUT /admin/stores
       └── GET /store/store
```

### Level 2 — StoreModuleService

```
StoreModuleService
  ├── retrieveStore()           (single-record fetch)
  ├── createStores()            (enforces single-record invariant)
  ├── updateStores()            (validates default_currency_code)
  ├── addStoreCurrencies()      (upsert into store_currency)
  └── removeStoreCurrencies()   (soft-delete from store_currency)
```

---

## 6. Runtime View

### Scenario: Add EUR Currency and Set as Default

1. `PUT /admin/stores/:id` with `{ default_currency_code: "eur", supported_currencies: [{ currency_code: "eur" }] }`.
2. `updateStores()` called.
3. `addStoreCurrencies([{ store_id, currency_code: "eur" }])` — upserts `store_currency` row.
4. Validates `"eur"` exists in `supported_currencies` list — passes.
5. Updates `store.default_currency_code = "eur"`.
6. `store.updated` event emitted.
7. Response: updated store DTO.

---

## 7. Deployment View

The Store module runs in-process. The `store` and `store_currency` tables are in the main PostgreSQL database. No separate service required.

---

## 8. Cross-Cutting Concerns

### Caching
`GET /store/store` responses are cacheable (CDN/reverse proxy). The store record changes infrequently; a `Cache-Control: max-age=60` header is appropriate.

### Event Emission
`store.created`, `store.updated`, `store.deleted` events propagate to the event bus, allowing other modules (e.g., Search index reconfig) to react to store changes.

---

## 9. Architecture Decisions

| ID  | Decision                                    | Rationale                                                         |
|-----|---------------------------------------------|-------------------------------------------------------------------|
| AD1 | Single store record enforced at DB level    | Unique partial index prevents races during concurrent setup calls |
| AD2 | Link Modules for region/sales channel links | Avoids coupling Store schema to Region/SalesChannel schemas       |
| AD3 | Separate `store_currency` join table        | Supports multiple currencies cleanly; avoids JSONB array pitfalls |
| AD4 | Storefront endpoint returns restricted fields | Prevents metadata leakage to public consumers                   |

---

## 10. Quality Scenarios

| Quality     | Scenario                                             | Measure                                   |
|-------------|------------------------------------------------------|-------------------------------------------|
| Integrity   | Two concurrent POST /admin/stores requests           | One succeeds, one returns 400             |
| Correctness | Set default currency not in supported list           | 400 returned; no partial update           |
| Performance | Storefront fetches store config on every page load   | < 5ms from DB; CDN cache reduces to < 1ms|
