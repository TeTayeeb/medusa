# Software Design Document — Fulfillment Module

**Module:** `@medusajs/fulfillment`  
**Version:** Medusa v2.15.4  
**Status:** Production  
**Last Updated:** 2025

---

## 1. Purpose and Scope

This document describes the internal design, data model, service interface, and provider integration model of the Fulfillment Module. The module is responsible for all aspects of physical goods delivery: geographic zone configuration, shipping option management, fulfillment record lifecycle, and label generation.

---

## 2. System Context

The Fulfillment Module is the logistics infrastructure layer of Medusa. It is invoked primarily by the Order module via core-flows workflows when line items need to be physically dispatched. It delegates carrier-specific operations to pluggable `IFulfillmentProvider` implementations.

```
Admin UI → createFulfillmentWorkflow → Fulfillment Module → IFulfillmentProvider (manual/carrier)
                                     ↗ ShippingOption config
Order Module → line items + address
```

---

## 3. Data Model

### 3.1 Entity Relationship Summary

| Entity | Table | PK Prefix | Notes |
|---|---|---|---|
| `FulfillmentSet` | `fulfillment_set` | `fuset_` | Top-level grouping |
| `ServiceZone` | `service_zone` | `serzo_` | Geographic area |
| `GeoZone` | `geo_zone` | `fgz_` | Specific location definition |
| `ShippingProfile` | `shipping_profile` | `sp_` | Product shipping category |
| `FulfillmentProvider` | `fulfillment_provider` | — | Provider registry entry |
| `ShippingOptionType` | `shipping_option_type` | `sotyp_` | Named delivery category |
| `ShippingOption` | `shipping_option` | `so_` | Purchasable delivery method |
| `ShippingOptionRule` | `shipping_option_rule` | `sorul_` | Conditional availability rule |
| `Fulfillment` | `fulfillment` | `ful_` | Shipment record |
| `FulfillmentItem` | `fulfillment_item` | `fulit_` | Item in a shipment |
| `FulfillmentLabel` | `fulfillment_label` | `fulla_` | Carrier label data |
| `FulfillmentAddress` | `fulfillment_address` | — | Snapshot of delivery address |

### 3.2 FulfillmentSet Fields

```
FulfillmentSet {
  id         : string (PK, prefix: fuset)
  name       : string (unique where deleted_at IS NULL)
  type       : string   -- e.g. "shipping", "pickup"
  metadata   : json | null
  deleted_at : datetime | null
}
```

Cascade delete → `service_zones` → `geo_zones` + `shipping_options`.

### 3.3 ServiceZone Fields

```
ServiceZone {
  id                 : string (PK, prefix: serzo)
  name               : string (unique where deleted_at IS NULL)
  fulfillment_set_id : FK → FulfillmentSet
  metadata           : json | null
  deleted_at         : datetime | null
}
```

Cascade delete → `geo_zones`, `shipping_options`.

### 3.4 GeoZone Fields

```
GeoZone {
  id                : string (PK, prefix: fgz)
  type              : GeoZoneType (country | province | city | zip)
  country_code      : string (ISO 3166-1 alpha-2)
  province_code     : string | null
  city              : string | null
  postal_expression : json | null  -- regex or range spec for zip matching
  service_zone_id   : FK → ServiceZone
  metadata          : json | null
  deleted_at        : datetime | null
}
```

Indexes on `country_code`, `province_code`, `city` enable fast address matching.

### 3.5 ShippingOption Fields

```
ShippingOption {
  id                     : string (PK, prefix: so)
  name                   : string (searchable, translatable)
  price_type             : ShippingOptionPriceType (flat_rate | calculated)
  data                   : json | null    -- provider config blob
  metadata               : json | null
  service_zone_id        : FK → ServiceZone
  shipping_profile_id    : FK → ShippingProfile (nullable)
  provider_id            : FK → FulfillmentProvider (nullable)
  shipping_option_type_id: FK → ShippingOptionType
  deleted_at             : datetime | null
}
```

Cascade delete → `rules`.

### 3.6 ShippingOptionRule Fields

```
ShippingOptionRule {
  id                 : string (PK, prefix: sorul)
  attribute          : string   -- e.g. "cart_total", "item_count"
  operator           : RuleOperator (eq | ne | gt | gte | lt | lte | in | nin)
  value              : json | null
  shipping_option_id : FK → ShippingOption
}
```

Rules on the same shipping option are combined with AND logic. The attribute names are convention-based strings evaluated by the pricing/cart layer.

### 3.7 Fulfillment Fields

```
Fulfillment {
  id                  : string (PK, prefix: ful)
  location_id         : string  -- cross-module ref to StockLocation
  packed_at           : datetime | null
  shipped_at          : datetime | null
  marked_shipped_by   : string | null
  created_by          : string | null
  delivered_at        : datetime | null
  canceled_at         : datetime | null
  data                : json | null  -- provider-returned data
  requires_shipping   : boolean (default: true)
  provider_id         : FK → FulfillmentProvider (nullable)
  shipping_option_id  : FK → ShippingOption (nullable)
  delivery_address_id : FK → FulfillmentAddress (nullable)
  metadata            : json | null
  deleted_at          : datetime | null
}
```

Cascade delete → `delivery_address`, `items`, `labels`.

### 3.8 FulfillmentItem Fields

```
FulfillmentItem {
  id                  : string (PK, prefix: fulit)
  title               : string     -- denormalized product title
  sku                 : string     -- denormalized SKU
  barcode             : string     -- denormalized barcode
  quantity            : bigNumber
  line_item_id        : string | null   -- cross-module ref to Order LineItem
  inventory_item_id   : string | null   -- cross-module ref to InventoryItem
  fulfillment_id      : FK → Fulfillment
  deleted_at          : datetime | null
}
```

### 3.9 FulfillmentLabel Fields

```
FulfillmentLabel {
  id              : string (PK, prefix: fulla)
  tracking_number : string
  tracking_url    : string
  label_url       : string
  fulfillment_id  : FK → Fulfillment
  deleted_at      : datetime | null
}
```

---

## 4. Service Layer

### 4.1 IFulfillmentModuleService

Key methods:

```typescript
interface IFulfillmentModuleService {
  // Configuration
  createFulfillmentSets(data, context?): Promise<FulfillmentSetDTO[]>
  createServiceZones(data, context?): Promise<ServiceZoneDTO[]>
  createGeoZones(data, context?): Promise<GeoZoneDTO[]>
  createShippingOptions(data, context?): Promise<ShippingOptionDTO[]>
  updateShippingOptions(data, context?): Promise<ShippingOptionDTO[]>

  // Fulfillment lifecycle
  createFulfillment(data, items, order, shippingOption, context?): Promise<FulfillmentDTO>
  cancelFulfillment(id, context?): Promise<FulfillmentDTO>
  createShipment(id, labels, context?): Promise<FulfillmentDTO>
  markFulfillmentAsDelivered(id, context?): Promise<FulfillmentDTO>

  // Provider validation
  validateFulfillmentData(providerId, optionData, data, context): Promise<Record<string, unknown>>
  validateShippingOption(providerId, data): Promise<boolean>
  canCalculate(providerId, data): Promise<boolean>
  calculatePrice(providerId, optionData, data, cart): Promise<CalculatedShippingOptionPrice>
  getFulfillmentOptions(providerId): Promise<Record<string, unknown>[]>

  // Retrieval
  listFulfillments(filters?, config?, context?): Promise<FulfillmentDTO[]>
  retrieveShippingOption(id, config?, context?): Promise<ShippingOptionDTO>
}
```

### 4.2 Provider Delegation

When `createFulfillment` is called:
1. The service resolves the `FulfillmentProvider` from the MedusaJS container
2. Calls `provider.createFulfillment(data, items, order, fulfillment)` asynchronously
3. Stores the returned `data` blob on the `Fulfillment` record (tracking info, carrier reference, etc.)
4. Provider errors cause full workflow compensation (rollback)

### 4.3 Lifecycle Timestamps

Fulfillment state is modeled through nullable timestamps rather than an enum status field:

| State | Condition |
|---|---|
| Created | `canceled_at IS NULL AND shipped_at IS NULL` |
| Packed | `packed_at IS NOT NULL AND shipped_at IS NULL` |
| Shipped | `shipped_at IS NOT NULL AND delivered_at IS NULL` |
| Delivered | `delivered_at IS NOT NULL` |
| Canceled | `canceled_at IS NOT NULL` |

---

## 5. Address Snapshot Pattern

`FulfillmentAddress` is a denormalized copy of the delivery address at the time of fulfillment creation. This ensures the fulfillment record remains accurate even if the customer later updates their address — delivery audit integrity is preserved without foreign keys to the customer address table.

---

## 6. Provider Registration

Fulfillment providers are registered in `medusa-config.ts`:

```typescript
{
  resolve: "@medusajs/fulfillment",
  options: {
    providers: [
      { resolve: "@medusajs/fulfillment-manual", id: "manual" },
      { resolve: "./fulfillment-fedex", id: "fedex",
        options: { apiKey: process.env.FEDEX_API_KEY }
      }
    ]
  }
}
```

The module stores a `FulfillmentProvider` record per registered provider and looks them up via the container when processing requests.

---

## 7. Known Constraints

- `ShippingOption.name` supports i18n via the `translatable()` marker — translations stored in `translation` module
- `FulfillmentSet.name` unique constraint prevents duplicate set names
- Cross-module references (`line_item_id`, `inventory_item_id`, `location_id`) are not enforced at DB level — integrity maintained at workflow layer
