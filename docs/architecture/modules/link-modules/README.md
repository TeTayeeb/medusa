# Link Modules

## Overview

Link Modules implement pseudo-foreign-key relationships between entities that live in separate Medusa modules. Because modules are designed to be isolated — each owning its own database schema — they cannot reference each other's tables with native SQL foreign keys. Link Modules solve this by introducing lightweight pivot tables that store only the two entity IDs and any relationship metadata.

This design preserves module independence while enabling rich cross-module data retrieval through Medusa's `Query` helper.

## Key Features

- **Module isolation preserved**: No direct FK constraints cross module boundaries.
- **Pivot table per relationship**: Each link is a dedicated table with `(entity_a_id, entity_b_id)`.
- **Query-time joins**: The `@medusajs/framework` Query helper resolves links transparently.
- **First-class link definitions**: Links are declared using `defineLink` from `@medusajs/framework`.
- **Bidirectional traversal**: Links can be queried from either side of the relationship.
- **Metadata support**: Optional `data` JSON column allows storing relationship-specific metadata (e.g., rank, flags).

## Defined Links

| Link Name                               | Entity A            | Entity B              |
|-----------------------------------------|---------------------|-----------------------|
| `ProductVariantInventoryItemLink`       | Product variant     | Inventory item        |
| `ProductSalesChannelLink`               | Product             | Sales channel         |
| `StockLocationSalesChannelLink`         | Stock location      | Sales channel         |
| `ApiKeySalesChannelLink`                | API key             | Sales channel         |
| `UserRoleLink`                          | User                | Role (RBAC)           |
| `OrderSalesChannelLink`                 | Order               | Sales channel         |
| `CartSalesChannelLink`                  | Cart                | Sales channel         |
| `StoreDefaultSalesChannelLink`          | Store               | Sales channel         |
| `StoreDefaultRegionLink`                | Store               | Region                |

## Link Definition Pattern

```typescript
// packages/core/core-flows/src/definitions/links/product-sales-channel.ts
import { defineLink } from "@medusajs/framework/utils"
import ProductModule from "@medusajs/product"
import SalesChannelModule from "@medusajs/sales-channel"

export default defineLink(
  ProductModule.linkable.product,
  SalesChannelModule.linkable.salesChannel
)
```

## Query Usage

```typescript
const query = container.resolve(ContainerRegistrationKeys.QUERY)

const { data: products } = await query.graph({
  entity: "product",
  fields: ["id", "title", "sales_channels.*"],
  filters: { sales_channels: { id: "sc_01" } },
})
```

## Pivot Table Schema

Each generated link table follows this structure:

| Column          | Type      | Description                           |
|-----------------|-----------|---------------------------------------|
| `id`            | string    | Unique link record ID                 |
| `{entity_a}_id` | string    | ID of the first entity                |
| `{entity_b}_id` | string    | ID of the second entity               |
| `data`          | JSON      | Optional relationship metadata        |
| `created_at`    | timestamp | Record creation time                  |
| `deleted_at`    | timestamp | Soft-delete timestamp                 |

## Module Registration

Link modules register automatically when the paired modules are both active. No explicit configuration is required for built-in links.

## Dependencies

| Dependency           | Purpose                                  |
|----------------------|------------------------------------------|
| `@medusajs/framework` | `defineLink`, Query helper, DB layer    |
| Paired modules       | Provide `linkable` entity descriptors    |
