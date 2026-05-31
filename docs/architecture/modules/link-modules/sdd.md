# Software Design Document — Link Modules

## 1. Purpose & Scope

This document describes the internal design of Medusa's Link Modules system (v2.15.4). It covers the pivot table generation mechanism, the `defineLink` API, Query helper integration, link lifecycle (create, delete, cascade), and the architectural rationale for avoiding cross-module foreign keys.

## 2. Architectural Rationale

Medusa modules are designed to be independently deployable. Each module owns its own schema namespace, and no module may reference another module's tables via SQL foreign keys. This constraint enables:

- Module hot-swap without schema migrations in unrelated modules.
- Independent horizontal scaling of module services.
- Clean domain boundaries for eventual service extraction.

Link Modules fulfil the need for relational queries across these boundaries by providing a lightweight pivot table per relationship, owned by neither module.

## 3. Link Definition API

```typescript
// Example: Product ↔ Sales Channel
import { defineLink } from "@medusajs/framework/utils"
import ProductModule from "@medusajs/product"
import SalesChannelModule from "@medusajs/sales-channel"

export default defineLink(
  ProductModule.linkable.product,
  SalesChannelModule.linkable.salesChannel,
  {
    database: {
      table: "product_sales_channel",
      idPrefix: "prodsc",
    },
  }
)
```

`defineLink` returns a `ModuleJoinerConfig` that the framework uses to:
1. Generate the pivot table DDL during `db:migrate`.
2. Register the link in the `RemoteLinkService` IoC binding.
3. Expose both directions of traversal in the Query helper's schema graph.

## 4. Generated Pivot Table Structure

```sql
CREATE TABLE product_sales_channel (
  id              VARCHAR  NOT NULL PRIMARY KEY,
  product_id      VARCHAR  NOT NULL,
  sales_channel_id VARCHAR NOT NULL,
  data            JSONB,
  created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  deleted_at      TIMESTAMP WITH TIME ZONE,
  UNIQUE (product_id, sales_channel_id)
);
CREATE INDEX ON product_sales_channel (product_id);
CREATE INDEX ON product_sales_channel (sales_channel_id);
```

The `data` JSONB column is available for relationship metadata (e.g., sort rank, visibility flags) without requiring a schema change.

## 5. RemoteLinkService API

```typescript
interface IRemoteLinkService {
  create(
    data: LinkDefinition | LinkDefinition[],
    sharedContext?: Context
  ): Promise<void>

  delete(
    data: LinkDefinition | LinkDefinition[],
    sharedContext?: Context
  ): Promise<void>

  softDelete(
    data: LinkDefinition | LinkDefinition[],
    sharedContext?: Context
  ): Promise<void>

  restore(
    data: LinkDefinition | LinkDefinition[],
    sharedContext?: Context
  ): Promise<void>

  dismiss(
    data: LinkDefinition | LinkDefinition[],
    sharedContext?: Context
  ): Promise<void>
}
```

## 6. Query Helper Integration

The `Query` helper (`@medusajs/framework`'s `ContainerRegistrationKeys.QUERY`) builds a virtual entity graph from all registered link definitions. When `query.graph()` resolves a field that crosses a module boundary, it:

1. Identifies the relevant link table from the joiner config.
2. Runs an IN query against the pivot table to retrieve related IDs.
3. Fetches the related entities from the target module service.
4. Merges the results into the response object.

```typescript
const { data } = await query.graph({
  entity: "product",
  fields: ["id", "title", "sales_channels.id", "sales_channels.name"],
  filters: { id: "prod_01" },
})
// → { id: "prod_01", title: "...", sales_channels: [{ id: "sc_01", name: "Web" }] }
```

## 7. Link Cascade Behaviour

When a parent entity is deleted:

1. The module service soft-deletes the entity (`deleted_at` set).
2. An event `{module}.{entity}.deleted` is emitted on the event bus.
3. The `RemoteLinkService` subscriber listens for deletion events and soft-deletes all pivot records where `{entity}_id` matches.

Hard deletes cascade to hard-delete pivot records.

## 8. Workflow Integration

Core-flows workflows that manage links follow a consistent pattern:

```typescript
// Step: create product-sales-channel links
export const attachProductsToSalesChannelStep = createStep(
  "attach-products-to-sales-channel",
  async ({ productIds, salesChannelId }, { container }) => {
    const remoteLink = container.resolve(ContainerRegistrationKeys.REMOTE_LINK)
    await remoteLink.create(
      productIds.map(id => ({
        [Modules.PRODUCT]:       { product_id: id },
        [Modules.SALES_CHANNEL]: { sales_channel_id: salesChannelId },
      }))
    )
    return new StepResponse(void 0, { productIds, salesChannelId })
  },
  async ({ productIds, salesChannelId }, { container }) => {
    const remoteLink = container.resolve(ContainerRegistrationKeys.REMOTE_LINK)
    await remoteLink.delete(
      productIds.map(id => ({
        [Modules.PRODUCT]:       { product_id: id },
        [Modules.SALES_CHANNEL]: { sales_channel_id: salesChannelId },
      }))
    )
  }
)
```

## 9. All Built-in Links Summary

| Table Name                           | Module A          | Module B          |
|--------------------------------------|-------------------|-------------------|
| `product_variant_inventory_item`     | PRODUCT           | INVENTORY         |
| `product_sales_channel`              | PRODUCT           | SALES_CHANNEL     |
| `stock_location_sales_channel`       | STOCK_LOCATION    | SALES_CHANNEL     |
| `publishable_api_key_sales_channel`  | API_KEY           | SALES_CHANNEL     |
| `user_role`                          | USER              | RBAC              |
| `order_sales_channel`                | ORDER             | SALES_CHANNEL     |
| `cart_sales_channel`                 | CART              | SALES_CHANNEL     |
| `store_default_sales_channel`        | STORE             | SALES_CHANNEL     |
| `store_default_region`               | STORE             | REGION            |

## 10. Error Handling

| Scenario                         | Behaviour                                          |
|----------------------------------|----------------------------------------------------|
| Duplicate link creation          | Upsert; existing record updated (no duplicate error)|
| Delete non-existent link         | No-op; no error thrown                             |
| Linked entity hard-deleted first | Pivot record orphaned; cleaned by maintenance job  |
