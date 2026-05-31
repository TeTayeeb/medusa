# Software Design Document — Inventory Module

**Module:** `@medusajs/inventory`  
**Version:** Medusa v2.15.4  
**Status:** Stable

---

## 1. Purpose and Scope

This SDD describes the internal design of the Inventory module: its data model (three entities with BigNumber quantities), the `InventoryModuleService` and its specialised `InventoryLevelService` sub-service, the custom repository, quantity computation rules, and domain events.

---

## 2. Data Model

### 2.1 Entity Relationship

```
InventoryItem ──────────── InventoryLevel
     │                    (location_id → StockLocation soft ref)
     └── ReservationItem
         (line_item_id → Order/Cart soft ref)
```

### 2.2 Entity Definitions

#### InventoryItem
```sql
CREATE TABLE inventory_item (
  id                TEXT PRIMARY KEY,  -- prefix: iitem_
  sku               TEXT,
  origin_country    TEXT,
  hs_code           TEXT,
  mid_code          TEXT,
  material          TEXT,
  weight            NUMERIC,
  length            NUMERIC,
  height            NUMERIC,
  width             NUMERIC,
  requires_shipping BOOLEAN DEFAULT TRUE,
  description       TEXT,
  title             TEXT,
  thumbnail         TEXT,
  metadata          JSONB,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at        TIMESTAMPTZ
);
CREATE UNIQUE INDEX ON inventory_item (sku) WHERE deleted_at IS NULL AND sku IS NOT NULL;
```

Virtual (computed) columns `reserved_quantity` and `stocked_quantity` are aggregated from related `InventoryLevel` rows and returned in read queries.

#### InventoryLevel
```sql
CREATE TABLE inventory_level (
  id                  TEXT PRIMARY KEY,  -- prefix: ilev_
  inventory_item_id   TEXT NOT NULL REFERENCES inventory_item(id),
  location_id         TEXT NOT NULL,
  stocked_quantity    NUMERIC DEFAULT 0,
  raw_stocked_quantity JSONB,
  reserved_quantity   NUMERIC DEFAULT 0,
  raw_reserved_quantity JSONB,
  incoming_quantity   NUMERIC DEFAULT 0,
  raw_incoming_quantity JSONB,
  metadata            JSONB,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);
CREATE UNIQUE INDEX ON inventory_level (inventory_item_id, location_id) WHERE deleted_at IS NULL;
CREATE INDEX IDX_inventory_level_inventory_item_id ON inventory_level (inventory_item_id) WHERE deleted_at IS NULL;
CREATE INDEX IDX_inventory_level_location_id ON inventory_level (location_id) WHERE deleted_at IS NULL;
```

`available_quantity` is a computed property: `stocked_quantity - reserved_quantity`. It is never stored; it is calculated on read.

#### ReservationItem
```sql
CREATE TABLE reservation_item (
  id                  TEXT PRIMARY KEY,  -- prefix: resitem_
  line_item_id        TEXT,
  inventory_item_id   TEXT NOT NULL REFERENCES inventory_item(id),
  location_id         TEXT NOT NULL,
  quantity            NUMERIC NOT NULL,
  raw_quantity        JSONB NOT NULL,
  allow_backorder     BOOLEAN DEFAULT FALSE,
  external_id         TEXT,
  description         TEXT,
  created_by          TEXT,
  metadata            JSONB,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at          TIMESTAMPTZ
);
CREATE INDEX IDX_reservation_item_line_item_id ON reservation_item (line_item_id) WHERE deleted_at IS NULL;
CREATE INDEX IDX_reservation_item_inventory_item_id ON reservation_item (inventory_item_id) WHERE deleted_at IS NULL;
CREATE INDEX IDX_reservation_item_location_id ON reservation_item (location_id) WHERE deleted_at IS NULL;
```

---

## 3. Service Architecture

### 3.1 Class Hierarchy

```
IInventoryService (interface)
  └── InventoryModuleService extends MedusaService<{...}>
        ├── inventoryItemService_    (IMedusaInternalService<InventoryItem>)
        ├── reservationItemService_  (IMedusaInternalService<ReservationItem>)
        └── inventoryLevelService_   (InventoryLevelService — specialised)

InventoryLevelService extends MedusaInternalService<InventoryLevel>
  └── Custom methods: adjustInventory, getAvailableQuantity, etc.
```

### 3.2 InventoryLevelService

`InventoryLevelService` extends the generated base with custom methods:

- **`adjustInventory(inventoryItemId, locationId, adjustment, ctx)`** — Atomically increments `stocked_quantity` by `adjustment` (negative values reduce stock). Uses `SELECT ... FOR UPDATE` to prevent race conditions.
- **`getAvailableQuantity(inventoryItemId, locationIds[], ctx)`** — Aggregates `stocked_quantity - reserved_quantity` across specified locations.
- **`getStockedQuantity(inventoryItemId, locationIds[], ctx)`** — Aggregates `stocked_quantity`.
- **`getReservedQuantity(inventoryItemId, locationIds[], ctx)`** — Aggregates `reserved_quantity`.

### 3.3 Reservation Flow

```
createReservationItems(input[])
  → for each input:
      1. ensureInventoryLevels([{ inventory_item_id, location_id }])
         → creates InventoryLevel if not exists
      2. confirmInventory(inventoryItemId, [locationId], quantity)
         → if available_quantity < quantity AND NOT allow_backorder → throw NOT_ALLOWED
      3. Create ReservationItem record
      4. Emit reservation_item.created
```

### 3.4 Inventory Adjustment Flow

```
adjustInventory(inventoryItemId, locationId, adjustment)
  → ensureInventoryLevels([{ inventory_item_id: inventoryItemId, location_id: locationId }])
  → inventoryLevelService_.adjustInventory(inventoryItemId, locationId, adjustment)
     → UPDATE inventory_level SET stocked_quantity = stocked_quantity + adjustment
       WHERE inventory_item_id = ? AND location_id = ? AND deleted_at IS NULL
  → Emit inventory_level.updated
```

---

## 4. BigNumber Handling

All quantity fields use Medusa's `BigNumber` wrapper which stores:
- `amount` as a `NUMERIC` column for SQL arithmetic
- `raw_amount` as a `JSONB` column preserving the original precision representation

This allows operations like `SUM` in SQL while preserving decimal precision for display and API responses.

---

## 5. Joiner Configuration

```ts
{
  serviceName: Modules.INVENTORY,
  alias: [
    { name: "inventory_item", args: { entity: "InventoryItem" } },
    { name: "inventory_level", args: { entity: "InventoryLevel" } },
    { name: "reservation_item", args: { entity: "ReservationItem" } },
  ],
  primaryKeys: ["id"],
  linkableKeys: { inventory_item_id: "InventoryItem" },
}
```

---

## 6. Domain Events

| Method | Event |
|---|---|
| `createInventoryItems` | `inventory_item.created` |
| `updateInventoryItems` | `inventory_item.updated` |
| `deleteInventoryItems` | `inventory_item.deleted` |
| `createInventoryLevels` | `inventory_level.created` |
| `updateInventoryLevels` | `inventory_level.updated` |
| `adjustInventory` | `inventory_level.updated` |
| `createReservationItems` | `reservation_item.created` |
| `updateReservationItems` | `reservation_item.updated` |
| `deleteReservationItems` | `reservation_item.deleted` |

---

## 7. Error Handling

| Condition | Error Type | Message |
|---|---|---|
| `available_quantity < requested && !allow_backorder` | `NOT_ALLOWED` | "Not enough stock available for item X at location Y" |
| InventoryLevel not found for deletion | `NOT_FOUND` | "Inventory level for item X at location Y not found" |
| InventoryItem not found | `NOT_FOUND` | "InventoryItem with id X not found" |

---

## 8. Migrations

| Migration | Change |
|---|---|
| `20240307132720` | Initial schema |
| `20240719123015` | BigNumber raw columns |
| `20241213063611` | Computed quantity indexes |
| `20251010131115` | Soft-delete adjustments |
