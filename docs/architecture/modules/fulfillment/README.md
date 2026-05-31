# Fulfillment Module

**Package:** `@medusajs/fulfillment`  
**Version:** Medusa v2.15.4  
**Module Key:** `Modules.FULFILLMENT`

## Overview

The Fulfillment Module handles the complete logistics lifecycle for a Medusa commerce application: from defining geographic service areas and shipping options, to creating fulfillment records, generating shipping labels, and tracking delivery milestones. It is provider-agnostic — shipping carriers and fulfillment services are plugged in via the `IFulfillmentProvider` interface — making it straightforward to integrate manual fulfillment, third-party carriers (FedEx, UPS, DHL), or custom warehouse management systems.

## Key Concepts

### Fulfillment Sets

A `FulfillmentSet` is the top-level configuration entity. It groups a collection of `ServiceZone` entries under a named, typed unit (e.g., `"default"`, `"express"`, `"pickup"`). Fulfillment sets are typically associated with stock locations via a module link, defining which shipping capabilities a warehouse or store supports.

### Service Zones

A `ServiceZone` belongs to exactly one `FulfillmentSet` and defines a logical geographic area. It collects:
- **GeoZones** — geographic definitions that determine *where* this service zone applies
- **ShippingOptions** — the delivery methods available within this zone

Service zone names must be unique across the system.

### Geo Zones

`GeoZone` entities represent geographic coverage rules with four granularity levels (controlled by `type`):

| Type | Fields Used |
|---|---|
| `country` | `country_code` |
| `province` | `country_code`, `province_code` |
| `city` | `country_code`, `province_code`, `city` |
| `zip` | `country_code`, `province_code`, `postal_expression` (JSON) |

A service zone may contain multiple geo zones, allowing complex geographic configurations (e.g., "all of Germany except Bavaria").

### Shipping Options

A `ShippingOption` represents a purchasable delivery method. Key fields:
- **price_type:** `flat_rate` (fixed price) or `calculated` (computed at runtime by the provider)
- **provider:** reference to a `FulfillmentProvider` (e.g., `manual`, `fedex`)
- **type:** reference to a `ShippingOptionType` (a named category, e.g., "Standard", "Express")
- **shipping_profile:** groups options by product compatibility (e.g., "Digital", "Fragile")
- **rules:** `ShippingOptionRule` entries that restrict when the option is available (e.g., cart weight limits, destination restrictions)
- **data:** provider-specific configuration blob passed to the fulfillment provider

### Shipping Option Rules

`ShippingOptionRule` entities conditionally enable/disable a shipping option. Each rule has:
- `attribute` — the cart/order attribute to test (e.g., `cart_total`, `item_count`, `shipping_address.country_code`)
- `operator` — comparison operator (`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`)
- `value` — JSON value to compare against

Multiple rules on an option are evaluated with AND logic.

### Fulfillment Providers

`FulfillmentProvider` is a registry entity linking a provider ID string (e.g., `manual`, `fedex_fulfillment`) to the system. Actual logic is implemented in provider packages that implement `IFulfillmentProvider`.

### Fulfillments

A `Fulfillment` record is created when items from an order are dispatched. It tracks:
- `location_id` — the stock location from which items are shipped
- `packed_at`, `shipped_at`, `delivered_at`, `canceled_at` — lifecycle timestamps
- `marked_shipped_by`, `created_by` — audit fields
- `requires_shipping` — whether physical shipment is involved
- `delivery_address` — a copied snapshot of the destination address
- `items` — the line items included in this shipment
- `labels` — generated shipping labels
- `data` — provider-specific fulfillment data

### Fulfillment Items

`FulfillmentItem` links a fulfillment to a specific line item from the originating order:
- `line_item_id` — foreign key to the order line item (cross-module, not a DB constraint)
- `inventory_item_id` — the inventory item being fulfilled
- `quantity` — amount being fulfilled
- `title`, `sku`, `barcode` — denormalized product data at time of fulfillment

### Fulfillment Labels

`FulfillmentLabel` stores shipping label data returned by the fulfillment provider:
- `tracking_number` — carrier tracking code
- `tracking_url` — link to carrier tracking page
- `label_url` — URL to the printable label PDF/image

## Key Workflows

| Workflow | Description |
|---|---|
| `createFulfillmentWorkflow` | Creates a fulfillment record and delegates to the provider |
| `cancelFulfillmentWorkflow` | Cancels a fulfillment, invoking provider cancellation |
| `createShipmentWorkflow` | Marks a fulfillment as shipped and creates labels |
| `markFulfillmentAsDeliveredWorkflow` | Records delivery timestamp |

## Admin API

| Endpoint | Description |
|---|---|
| `GET /admin/fulfillment-sets` | List fulfillment sets |
| `POST /admin/fulfillment-sets` | Create a fulfillment set |
| `DELETE /admin/fulfillment-sets/:id` | Delete a fulfillment set |
| `POST /admin/fulfillment-sets/:id/service-zones` | Add service zones |
| `GET /admin/service-zones/:id` | Get service zone |
| `POST /admin/service-zones/:id/geo-zones` | Add geo zones |
| `GET /admin/shipping-options` | List shipping options |
| `POST /admin/shipping-options` | Create shipping option |
| `POST /admin/fulfillments` | Create a fulfillment |
| `POST /admin/fulfillments/:id/shipment` | Create shipment for fulfillment |
| `POST /admin/fulfillments/:id/cancel` | Cancel a fulfillment |

## Module Links

- **Stock Location ↔ Fulfillment Set** — associates a location with fulfillment capabilities
- **Order ↔ Fulfillment** — links fulfillment records to order line items
- **Sales Channel ↔ Fulfillment Set** — defines which shipping options are available per channel

## Provider Interface

```typescript
interface IFulfillmentProvider {
  getFulfillmentOptions(): Promise<Record<string, unknown>[]>
  validateFulfillmentData(data, itemData, context): Promise<Record<string, unknown>>
  validateOption(data): Promise<boolean>
  canCalculate(data): Promise<boolean>
  calculatePrice(optionData, data, cart): Promise<number>
  createFulfillment(data, items, order, fulfillment): Promise<Record<string, unknown>>
  cancelFulfillment(data): Promise<Record<string, unknown>>
  createReturnFulfillment(fromData): Promise<Record<string, unknown>>
  createShipment(data, items, context): Promise<{ tracking_links, data }>
}
```
