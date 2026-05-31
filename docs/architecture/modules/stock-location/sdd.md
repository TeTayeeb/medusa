# Software Design Document — Stock Location Module

**Module:** `@medusajs/stock-location`  
**Version:** Medusa v2.15.4  
**Status:** Stable

---

## 1. Purpose and Scope

This SDD describes the internal design of the Stock Location module: its two-entity data model, the `StockLocationModuleService` class, the event-bus integration, and the upsert pattern used for address management.

---

## 2. Data Model

### 2.1 Entity Relationship

```
StockLocationAddress ──1:1── StockLocation
```

The address owns the relationship: `StockLocation.address_id` references `StockLocationAddress.id`. Deleting a `StockLocationAddress` cascades to the related `StockLocation`.

### 2.2 Entity Definitions

#### StockLocation
```sql
CREATE TABLE stock_location (
  id         TEXT PRIMARY KEY,   -- prefix: sloc_
  name       TEXT NOT NULL,
  address_id TEXT REFERENCES stock_location_address(id),
  metadata   JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at TIMESTAMPTZ
);
CREATE INDEX ON stock_location (name) WHERE deleted_at IS NULL;
```

#### StockLocationAddress
```sql
CREATE TABLE stock_location_address (
  id           TEXT PRIMARY KEY,  -- prefix: laddr_
  address_1    TEXT NOT NULL,
  address_2    TEXT,
  company      TEXT,
  city         TEXT,
  country_code TEXT NOT NULL,
  phone        TEXT,
  province     TEXT,
  postal_code  TEXT,
  metadata     JSONB,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ
);
CREATE INDEX ON stock_location_address (country_code) WHERE deleted_at IS NULL;
```

### 2.3 Address Cascade

When a `StockLocationAddress` is deleted, its associated `StockLocation` is also deleted (cascades defined in the DML model). This ensures no orphaned location records exist without an address when the address is the canonical record.

---

## 3. Service Architecture

### 3.1 Class Hierarchy

```
IStockLocationService (interface)
  └── StockLocationModuleService extends MedusaService<{...}>
        ├── stockLocationService_        (IMedusaInternalService<StockLocation>)
        ├── stockLocationAddressService_ (IMedusaInternalService<StockLocationAddress>)
        └── eventBusModuleService_       (IEventBusService — optional)
```

### 3.2 Dependency Injection

| Dependency | Key | Required | Purpose |
|---|---|---|---|
| `EVENT_BUS` | `Modules.EVENT_BUS` | No | Emit domain events |
| `baseRepository` | `baseRepository` | Yes | ORM unit-of-work |
| `stockLocationService` | `stockLocationService` | Yes | CRUD for `StockLocation` |
| `stockLocationAddressService` | `stockLocationAddressService` | Yes | CRUD for `StockLocationAddress` |

The Event Bus is optional; the module degrades gracefully when none is configured (useful in test environments).

### 3.3 createStockLocations Flow

```
createStockLocations(data[])
  → for each item in data:
      1. If data.address is provided (inline):
         → stockLocationAddressService_.create(data.address)
         → set location.address_id = created address id
      2. Else if data.address_id is provided:
         → verify address exists
         → set location.address_id = data.address_id
      3. stockLocationService_.create(location)
  → Emit stock-location.created for each created id
  → Return StockLocationDTO[]
```

### 3.4 updateStockLocations Flow

```
updateStockLocations(idOrSelector, input)
  → Resolve location(s) matching id or selector
  → If input.address is provided:
      → If location.address_id exists:
          → stockLocationAddressService_.update(location.address_id, input.address)
      → Else:
          → stockLocationAddressService_.create(input.address)
          → update location.address_id = new address id
  → stockLocationService_.update(id, remaining_fields)
  → Emit stock-location.updated
```

### 3.5 upsertStockLocations

`upsertStockLocations` is a composite operation that calls either `create` or `update` depending on whether the provided ID/name already exists. This is used by admin import workflows.

---

## 4. Repository Layer

Both `StockLocation` and `StockLocationAddress` use the standard `MedusaInternalService` base without custom repository extensions. Queries support:

- Filtering by `name`, `address.city`, `address.country_code`
- Pagination via `skip` / `take`
- Field selection and relation expansion (`address`)
- Soft delete / restore

---

## 5. Joiner Configuration

```ts
{
  serviceName: Modules.STOCK_LOCATION,
  alias: [
    { name: "stock_location", args: { entity: "StockLocation" } },
    { name: "stock_location_address", args: { entity: "StockLocationAddress" } },
  ],
  primaryKeys: ["id"],
  linkableKeys: { stock_location_id: "StockLocation" },
}
```

This allows the Inventory module and other consumers to join location data via remote query using `stock_location_id` as the link key.

---

## 6. Domain Events

| Method | Event constant | Payload |
|---|---|---|
| `createStockLocations` | `stock-location.created` | `{ id }[]` |
| `updateStockLocations` | `stock-location.updated` | `{ id }[]` |
| `deleteStockLocations` | `stock-location.deleted` | `{ id }[]` |

Events are emitted after the transaction commits, ensuring downstream consumers only react to persisted state.

---

## 7. Validation

- `country_code` is required on `StockLocationAddress` (enforced at DB NOT NULL level).
- `address_1` is required on `StockLocationAddress`.
- `name` is required on `StockLocation` (NOT NULL, searchable).
- No uniqueness constraint on `name` — duplicate location names are allowed.

---

## 8. Migrations

| Migration | Change |
|---|---|
| `20240307161216` | Initial schema: `stock_location`, `stock_location_address` |
| `20241210073813` | Add `metadata` to address |
| `20250106142624` | Soft-delete index improvements |
| `20250120110820` | Searchable field adjustments |
