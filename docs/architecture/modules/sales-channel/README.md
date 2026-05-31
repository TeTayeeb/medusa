# Sales Channel Module

The Sales Channel module (`@medusajs/sales-channel`) enables multi-channel commerce in Medusa. A sales channel represents a distinct point of sale or customer-facing surface — a web storefront, mobile app, POS terminal, B2B portal, or marketplace listing. Product availability and stock locations can be scoped per sales channel, enabling granular control over what is sold where.

## Purpose

Modern commerce requires selling through multiple channels simultaneously. The Sales Channel module provides the coordination layer: it does not duplicate products or inventory, but instead defines which products and stock locations are accessible on each channel, and authenticates channel access via publishable API keys.

## Key Features

- **Multi-Channel Commerce** — Model any number of sales channels, each with its own product catalogue and inventory visibility.
- **Product Scoping** — Products are linked to sales channels through a module link (`product ↔ sales-channel`), so only assigned products appear on a given channel.
- **Stock Location Association** — Stock locations are linked to sales channels to control inventory availability per channel.
- **Publishable API Keys** — Each storefront authenticates its channel context by passing a publishable API key header (`x-publishable-api-key`), which resolves to a sales channel automatically.
- **Default Channel** — A store always has a default sales channel used when no explicit channel is specified.
- **Disable Flag** — Channels can be disabled without deletion, halting sales on that surface.

## Entities

| Entity         | Key Fields                                                        |
|----------------|-------------------------------------------------------------------|
| `SalesChannel` | `id`, `name`, `description`, `is_disabled`, `metadata`           |

## Admin API

| Method | Endpoint                      | Description                         |
|--------|-------------------------------|-------------------------------------|
| GET    | `/admin/sales-channels`       | List sales channels                 |
| POST   | `/admin/sales-channels`       | Create a sales channel              |
| GET    | `/admin/sales-channels/:id`   | Retrieve a sales channel            |
| POST   | `/admin/sales-channels/:id`   | Update a sales channel              |
| DELETE | `/admin/sales-channels/:id`   | Delete a sales channel              |

## Store API

The Store API does not expose sales channel endpoints directly. The active channel is resolved via the `x-publishable-api-key` request header. All store queries (product listings, cart creation, etc.) are automatically scoped to the resolved channel.

```http
GET /store/products
x-publishable-api-key: pk_01HXXXXXXXXXXXXXXXXXXXXX
```

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.SALES_CHANNEL
```

## Service Usage

```ts
const salesChannelService = container.resolve(Modules.SALES_CHANNEL)

// Create a channel
const channel = await salesChannelService.createSalesChannels({
  name: "Web Storefront",
  description: "Primary online store",
})

// Disable a channel
await salesChannelService.updateSalesChannels(channel.id, {
  is_disabled: true,
})

// List all active channels
const channels = await salesChannelService.listSalesChannels({
  is_disabled: false,
})
```

## Module Links

| Link                              | Description                                       |
|-----------------------------------|---------------------------------------------------|
| `product ↔ sales-channel`         | Controls which products are available per channel |
| `stock-location ↔ sales-channel`  | Controls which locations serve each channel       |
| `publishable-api-key ↔ sales-channel` | Maps API keys to channels                    |

## Related Modules

- **Product Module** — Products are linked to sales channels via the module link layer.
- **Inventory / Stock Location Module** — Stock locations are associated to channels for fulfillment routing.
- **API Key Module** — Publishable API keys resolve to one or more sales channels.
- **Cart Module** — Carts carry a `sales_channel_id` to scope the purchase context.

## Version

Medusa v2.15.4
