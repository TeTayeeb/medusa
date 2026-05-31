# SpecKit — Inventory Module

**Module:** `@medusajs/inventory`  
**Version:** Medusa v2.15.4  
**Document type:** Functional & Technical Specification

---

## 1. Functional Specifications

### 1.1 Inventory Items

**F-INV-01: Create Inventory Item**
- `sku` is optional but must be unique (among non-deleted items) when provided.
- Physical dimension fields (`weight`, `length`, `height`, `width`) are optional floats.
- `requires_shipping` defaults to `true`; set to `false` for digital/downloadable items.

**F-INV-02: Update Inventory Item**
- All fields are patchable post-creation.
- Changing `sku` must re-validate uniqueness.

**F-INV-03: Delete Inventory Item**
- Soft-deletes the item.
- Cascades soft-delete to all linked `InventoryLevel` and `ReservationItem` records.

### 1.2 Inventory Levels

**F-LEVEL-01: Create Inventory Level**
- Requires `inventory_item_id` and `location_id`.
- `(inventory_item_id, location_id)` must be unique among non-deleted levels.
- Initial `stocked_quantity` defaults to 0; can be set on creation.

**F-LEVEL-02: Adjust Inventory**
- `adjustInventory(inventoryItemId, locationId, adjustment)`:
  - `adjustment > 0` — stock received.
  - `adjustment < 0` — stock removed (e.g. after fulfillment confirmation, damage write-off).
- Result: `stocked_quantity += adjustment`. `reserved_quantity` is unchanged.
- Negative `stocked_quantity` is allowed for write-offs but triggers a warning.

**F-LEVEL-03: Computed available_quantity**
- `available_quantity = stocked_quantity - reserved_quantity` (computed, never stored).
- `available_quantity` can be negative only for items with active backorder reservations.

**F-LEVEL-04: Delete Inventory Level**
- Soft-deletes the level record.
- Does NOT delete active `ReservationItem` records referencing the same `(item, location)`.

### 1.3 Reservations

**F-RES-01: Create Reservation**
- Requires `inventory_item_id`, `location_id`, and `quantity`.
- If `available_quantity < quantity` AND `allow_backorder = false` → throw `NOT_ALLOWED`.
- `line_item_id` is optional (used for order/cart linking).
- Multiple reservations can exist for the same `(inventory_item_id, location_id)`.

**F-RES-02: Update Reservation**
- `quantity` can be reduced or increased (re-validates availability for increases).
- `location_id` can be changed (re-validates availability at new location).

**F-RES-03: Delete Reservation**
- Soft-deletes the reservation record.
- Does NOT automatically adjust `stocked_quantity` — stock was never deducted; the hold is simply released.

**F-RES-04: Delete by Line Item**
- `deleteReservationItemsByLineItem(lineItemId)` — bulk deletes all reservations for a line item.
- Used in order cancellation workflows.

---

## 2. Business Rules

| ID | Rule | Enforcement |
|---|---|---|
| BR-01 | `available_quantity = stocked_quantity - reserved_quantity` | Computed on read |
| BR-02 | Cannot reserve more than `available_quantity` unless `allow_backorder = true` | Service-layer check in `createReservationItems_` |
| BR-03 | SKU must be unique among non-deleted inventory items | DB partial unique index |
| BR-04 | `(inventory_item_id, location_id)` must be unique per non-deleted level | DB partial unique index |
| BR-05 | Deleting InventoryItem cascades to levels and reservations | DML cascade |
| BR-06 | Reservation delete does NOT adjust stock quantity | By design — reservation is a hold, not a deduction |
| BR-07 | `stocked_quantity` is only changed by `adjustInventory` or direct level update | Convention enforced by service |

---

## 3. API Contracts

### 3.1 POST /admin/inventory-items

**Request:**
```json
{
  "sku": "WIDGET-001",
  "title": "Blue Widget",
  "description": "A very blue widget",
  "requires_shipping": true,
  "weight": 250,
  "length": 10,
  "width": 5,
  "height": 3
}
```

**Response `200`:**
```json
{
  "inventory_item": {
    "id": "iitem_01EXAMPLE",
    "sku": "WIDGET-001",
    "title": "Blue Widget",
    "requires_shipping": true,
    "weight": 250,
    "stocked_quantity": 0,
    "reserved_quantity": 0
  }
}
```

### 3.2 POST /admin/inventory-items/:id/location-levels

**Request:**
```json
{
  "location_id": "sloc_01",
  "stocked_quantity": 100,
  "incoming_quantity": 50
}
```

**Response `200`:**
```json
{
  "inventory_level": {
    "id": "ilev_01",
    "inventory_item_id": "iitem_01",
    "location_id": "sloc_01",
    "stocked_quantity": 100,
    "reserved_quantity": 0,
    "incoming_quantity": 50,
    "available_quantity": 100
  }
}
```

### 3.3 POST /admin/reservations

**Request:**
```json
{
  "inventory_item_id": "iitem_01",
  "location_id": "sloc_01",
  "quantity": 2,
  "line_item_id": "li_01",
  "allow_backorder": false
}
```

**Response `200`:**
```json
{
  "reservation": {
    "id": "resitem_01",
    "inventory_item_id": "iitem_01",
    "location_id": "sloc_01",
    "quantity": 2,
    "line_item_id": "li_01",
    "allow_backorder": false
  }
}
```

**Errors:**
- `400 Bad Request` — `quantity` ≤ 0
- `400 Not Allowed` — insufficient available stock and `allow_backorder = false`

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `sku` | String, max 255 chars, unique (non-deleted) |
| `quantity` (reservation) | BigNumber > 0 |
| `stocked_quantity` | BigNumber ≥ 0 on creation; can go negative via adjustments |
| `incoming_quantity` | BigNumber ≥ 0 |
| `location_id` | Non-empty string |
| `inventory_item_id` | Must reference existing non-deleted InventoryItem |

---

## 5. Quantity Query API

```ts
// Returns sum of available_quantity across specified locations
retrieveAvailableQuantity(
  inventoryItemId: string,
  locationIds: string[]
): Promise<BigNumber>

// Returns true if available_quantity ≥ quantity
confirmInventory(
  inventoryItemId: string,
  locationIds: string[],
  quantity: BigNumberInput
): Promise<boolean>
```

---

## 6. Integration Test Scenarios

| Scenario | Expected |
|---|---|
| Create item → create level → create reservation | Reservation created, `available_quantity` decreases |
| Reserve more than available (no backorder) | `NOT_ALLOWED` error |
| Reserve more than available (backorder=true) | Reservation created |
| Adjust inventory by -10 | `stocked_quantity` decreases by 10 |
| Delete reservation | `available_quantity` increases by reservation qty |
| Delete item | All levels and reservations also soft-deleted |
| SKU duplicate | 409 / uniqueness error |
