# Inventory Module

**Package:** `@medusajs/inventory`  
**Module key:** `Modules.INVENTORY`  
**Version:** Medusa v2.15.4

---

## Overview

The Inventory module tracks physical stock availability across one or more warehouse locations. It provides three core capabilities:

1. **Inventory items** — abstract stockable units (typically linked to product variants via Module Links)
2. **Inventory levels** — quantity records per `(inventory_item, location)` pair, tracking stocked, reserved, incoming, and computed available quantities
3. **Reservation items** — transient holds placed against inventory during the checkout lifecycle, automatically reconciled on order completion or cancellation

The module enforces the fundamental invariant:

```
available_quantity = stocked_quantity - reserved_quantity
```

All quantity fields use Medusa's `BigNumber` type to support arbitrary precision and avoid floating-point rounding errors for high-volume operations.

---

## Key Entities

| Entity | DB table | ID prefix | Description |
|---|---|---|---|
| `InventoryItem` | `inventory_item` | `iitem_` | Represents a stockable SKU with physical attributes |
| `InventoryLevel` | `inventory_level` | `ilev_` | Quantity record at a specific stock location |
| `ReservationItem` | `reservation_item` | `resitem_` | A temporary hold on inventory for a line item |

### InventoryItem Fields
`id`, `sku` (searchable, nullable), `origin_country`, `hs_code`, `mid_code`, `material`, `weight`, `length`, `height`, `width`, `requires_shipping` (default true), `description`, `title`, `thumbnail`, `metadata`, `location_levels` (hasMany), `reservation_items` (hasMany), `reserved_quantity` (computed), `stocked_quantity` (computed).

### InventoryLevel Fields
`id`, `location_id` (text reference to StockLocation), `stocked_quantity` (BigNumber, default 0), `reserved_quantity` (BigNumber, default 0), `incoming_quantity` (BigNumber, default 0), `available_quantity` (computed BigNumber), `metadata`, `inventory_item` (belongsTo).

### ReservationItem Fields
`id`, `line_item_id` (nullable — references cart/order line item), `allow_backorder` (boolean), `location_id`, `quantity` (BigNumber), `raw_quantity` (JSON), `external_id` (nullable), `description`, `created_by`, `metadata`, `inventory_item` (belongsTo).

---

## Key Service Methods

```ts
// Inventory item lifecycle
createInventoryItems(input): Promise<InventoryItemDTO | InventoryItemDTO[]>
updateInventoryItems(input): Promise<InventoryItemDTO | InventoryItemDTO[]>
deleteInventoryItems(ids: string[]): Promise<void>
listInventoryItems(filters?, config?): Promise<InventoryItemDTO[]>

// Inventory level management
createInventoryLevels(input): Promise<InventoryLevelDTO | InventoryLevelDTO[]>
updateInventoryLevels(updates): Promise<InventoryLevelDTO | InventoryLevelDTO[]>
adjustInventory(
  inventoryItemId: string,
  locationId: string,
  adjustment: BigNumberInput   // positive = stock in, negative = stock out
): Promise<InventoryLevelDTO>
deleteInventoryLevel(inventoryItemId: string, locationId: string): Promise<void>
deleteInventoryItemLevelByLocationId(locationId: string | string[]): Promise<void>

// Reservation lifecycle
createReservationItems(input): Promise<ReservationItemDTO | ReservationItemDTO[]>
updateReservationItems(id | selector, data): Promise<ReservationItemDTO | ReservationItemDTO[]>
deleteReservationItems(ids: string[]): Promise<void>
deleteReservationItemsByLineItem(lineItemId: string | string[]): Promise<void>

// Availability queries
retrieveAvailableQuantity(inventoryItemId, locationIds: string[]): Promise<BigNumber>
retrieveStockedQuantity(inventoryItemId, locationIds: string[]): Promise<BigNumber>
retrieveReservedQuantity(inventoryItemId, locationIds: string[]): Promise<BigNumber>
confirmInventory(inventoryItemId, locationIds, quantity): Promise<boolean>
```

---

## Business Rules

| Rule | Description |
|---|---|
| `available_quantity ≥ 0` | Cannot reserve more than available unless `allow_backorder = true` |
| Reservation atomicity | `createReservationItems` checks and decrements in a single transaction |
| Location scoping | All quantity queries are scoped to one or more `location_id` values |
| Cascade deletes | Deleting an `InventoryItem` cascades to its levels and reservations |
| SKU uniqueness | `sku` is unique where `deleted_at IS NULL` |

---

## API Endpoints

### Admin API
| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/inventory-items` | List inventory items |
| `POST` | `/admin/inventory-items` | Create an inventory item |
| `GET` | `/admin/inventory-items/:id` | Retrieve an inventory item |
| `POST` | `/admin/inventory-items/:id` | Update an inventory item |
| `DELETE` | `/admin/inventory-items/:id` | Delete an inventory item |
| `GET` | `/admin/inventory-items/:id/location-levels` | List inventory levels for item |
| `POST` | `/admin/inventory-items/:id/location-levels` | Create an inventory level |
| `POST` | `/admin/inventory-items/:id/location-levels/:loc_id` | Update an inventory level |
| `DELETE` | `/admin/inventory-items/:id/location-levels/:loc_id` | Delete an inventory level |
| `GET` | `/admin/reservations` | List reservations |
| `POST` | `/admin/reservations` | Create a reservation |
| `POST` | `/admin/reservations/:id` | Update a reservation |
| `DELETE` | `/admin/reservations/:id` | Delete a reservation |

---

## Module Links

- **Product module** → `ProductVariant` linked to `InventoryItem` via link table `product_variant_inventory_item`
- **Stock-Location module** → `InventoryLevel.location_id` references `StockLocation.id` (soft reference, no DB FK)
- **Order module** → `ReservationItem.line_item_id` references order line items
- **Fulfillment module** → reads inventory availability before fulfillment creation

---

## Events Emitted

| Event | Payload | Trigger |
|---|---|---|
| `inventory_item.created` | `{ id }` | After `createInventoryItems` |
| `inventory_item.updated` | `{ id }` | After `updateInventoryItems` |
| `inventory_item.deleted` | `{ id }` | After `deleteInventoryItems` |
| `inventory_level.created` | `{ id }` | After `createInventoryLevels` |
| `inventory_level.updated` | `{ id }` | After `updateInventoryLevels` |
| `reservation_item.created` | `{ id }` | After `createReservationItems` |
| `reservation_item.deleted` | `{ id }` | After `deleteReservationItems` |

---

## Related Modules

- **Stock-Location** — provides the location entities referenced by `location_id`
- **Product** — variants link to inventory items
- **Cart/Order** — drive reservation creation/deletion workflows
- **Fulfillment** — confirms stock before creating shipments
