# Stock Location Module

**Package:** `@medusajs/stock-location`  
**Module key:** `Modules.STOCK_LOCATION`  
**Version:** Medusa v2.15.4

---

## Overview

The Stock Location module manages the physical and virtual locations from which a merchant fulfills orders. A stock location represents a warehouse, a third-party logistics provider (3PL) node, a retail store back-room, or any addressable point from which inventory is shipped.

The module is intentionally narrow in scope: it stores location names, addresses, and metadata. All quantity tracking is the responsibility of the **Inventory module**, and the assignment of locations to sales channels is handled via **Module Links** rather than direct foreign keys.

---

## Key Entities

| Entity | DB table | ID prefix | Description |
|---|---|---|---|
| `StockLocation` | `stock_location` | `sloc_` | A named fulfillment location (physical or virtual) |
| `StockLocationAddress` | `stock_location_address` | `laddr_` | Physical postal address associated with a location |

### StockLocation Fields
`id`, `name` (searchable), `metadata` (JSON nullable), `address` (belongsTo `StockLocationAddress`, nullable), `address_id` (FK nullable).

Timestamps: `created_at`, `updated_at`, `deleted_at` (soft delete).

### StockLocationAddress Fields
`id`, `address_1` (searchable), `address_2` (nullable), `company` (nullable), `city` (searchable, nullable), `country_code` (searchable), `phone` (nullable), `province` (searchable, nullable), `postal_code` (searchable, nullable), `metadata` (JSON nullable), `stock_locations` (hasOne back-reference).

Timestamps: `created_at`, `updated_at`, `deleted_at`.

---

## Key Service Methods

```ts
// Stock location lifecycle
createStockLocations(
  data: CreateStockLocationInput | CreateStockLocationInput[],
  context?: Context
): Promise<StockLocationDTO | StockLocationDTO[]>

upsertStockLocations(
  data: UpsertStockLocationInput | UpsertStockLocationInput[],
  context?: Context
): Promise<StockLocationDTO | StockLocationDTO[]>

updateStockLocations(
  id: string | FilterableStockLocationProps,
  input: UpdateStockLocationInput,
  context?: Context
): Promise<StockLocationDTO | StockLocationDTO[]>

deleteStockLocations(ids: string[], context?: Context): Promise<void>

listStockLocations(
  filters?: FilterableStockLocationProps,
  config?: FindConfig<StockLocationDTO>,
  context?: Context
): Promise<StockLocationDTO[]>

retrieveStockLocation(
  id: string,
  config?: FindConfig<StockLocationDTO>,
  context?: Context
): Promise<StockLocationDTO>

// Address management
upsertStockLocationAddresses(
  data: UpsertStockLocationAddressInput | UpsertStockLocationAddressInput[],
  context?: Context
): Promise<StockLocationAddressDTO | StockLocationAddressDTO[]>
```

### CreateStockLocationInput Shape

```ts
{
  name: string
  address?: StockLocationAddressInput  // inline address creation
  address_id?: string                  // or reference existing address
  metadata?: Record<string, unknown>
}
```

---

## API Endpoints

### Admin API
| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/stock-locations` | List stock locations (filterable, paginated) |
| `POST` | `/admin/stock-locations` | Create a new stock location |
| `GET` | `/admin/stock-locations/:id` | Retrieve a stock location |
| `POST` | `/admin/stock-locations/:id` | Update a stock location |
| `DELETE` | `/admin/stock-locations/:id` | Delete a stock location |

> Stock locations have no dedicated store-facing API; they are opaque to customers.

---

## Module Links

The Stock Location module integrates with the rest of the Medusa ecosystem via the **Module Link** layer:

| Link | Direction | Description |
|---|---|---|
| `SalesChannel ↔ StockLocation` | M:N via `sales_channel_stock_location` | Determines which locations serve which channels |
| `StockLocation ↔ FulfillmentProvider` | M:N via `location_fulfillment_provider` | Determines which fulfillment providers are available at a location |
| `InventoryLevel.location_id` | soft ref → `StockLocation.id` | Inventory levels reference location IDs (no DB FK) |

### Creating a Sales Channel → Location Link

```ts
import { createStep } from "@medusajs/framework/workflows-sdk"
import { ContainerRegistrationKeys, Modules } from "@medusajs/framework/utils"

// Via the remote link service:
remoteLink.create({
  [Modules.SALES_CHANNEL]: { sales_channel_id: "sc_01" },
  [Modules.STOCK_LOCATION]: { stock_location_id: "sloc_01" },
})
```

---

## Events Emitted

The Stock Location module emits the following domain events via the Event Bus:

| Event | Payload | Trigger |
|---|---|---|
| `stock-location.created` | `{ id }` | After `createStockLocations` |
| `stock-location.updated` | `{ id }` | After `updateStockLocations` |
| `stock-location.deleted` | `{ id }` | After `deleteStockLocations` |

---

## Configuration

```ts
// medusa-config.ts
import { Modules } from "@medusajs/framework/utils"

export default defineConfig({
  modules: [
    { resolve: "@medusajs/stock-location", key: Modules.STOCK_LOCATION }
  ]
})
```

---

## Related Modules

- **Inventory** — `InventoryLevel` entities reference `StockLocation.id` as `location_id`
- **Sales Channel** — locations are assigned to channels to control order routing
- **Fulfillment** — fulfillment providers are mapped to locations for shipping-option availability
- **Order** — fulfillment sets reference the originating stock location
