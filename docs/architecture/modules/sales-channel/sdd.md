# Software Design Document — Sales Channel Module

## 1. Overview

The Sales Channel module enables multi-channel commerce by defining distinct sales surfaces and controlling product/inventory visibility per channel. The module itself is intentionally lightweight — it stores channel definitions and delegates cross-module concerns (product assignment, inventory scoping, API key resolution) to the module link layer.

## 2. Goals and Non-Goals

**Goals:**
- Provide a registry of named sales channels.
- Allow enabling/disabling channels without deletion.
- Support association of products, stock locations, and publishable API keys through module links.
- Enable resolution of a channel context from an incoming API request (via publishable key).
- Maintain a default sales channel for stores that don't specify an explicit channel.

**Non-Goals:**
- Product catalogue management (delegated to Product module).
- Inventory management (delegated to Inventory/Stock Location module).
- API key creation or validation (delegated to API Key module).
- Price list scoping per channel (handled via Pricing module rules).

## 3. Data Model

### 3.1 SalesChannel Entity

```ts
@Entity()
export class SalesChannel extends BaseEntity {
  @PrimaryKey()
  id: string  // ULID

  @Property()
  name: string

  @Property({ nullable: true })
  description?: string

  @Property({ default: false })
  is_disabled: boolean

  @Property({ nullable: true, type: "jsonb" })
  metadata?: Record<string, unknown>

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.2 Database Table

| Table           | Primary Key | Notable Indexes     |
|-----------------|-------------|---------------------|
| `sales_channel` | `id` (ULID) | `deleted_at`, `name` |

The module maintains no foreign key relationships internally. All associations (product ↔ channel, location ↔ channel, api-key ↔ channel) are stored in pivot tables managed by the module link infrastructure.

## 4. Service Interface

```ts
interface ISalesChannelModuleService {
  createSalesChannels(data: CreateSalesChannelDTO[], sharedContext?: Context): Promise<SalesChannelDTO[]>
  updateSalesChannels(data: UpdateSalesChannelDTO[], sharedContext?: Context): Promise<SalesChannelDTO[]>
  deleteSalesChannels(ids: string[], sharedContext?: Context): Promise<void>
  retrieveSalesChannel(id: string, config?: FindConfig<SalesChannelDTO>, sharedContext?: Context): Promise<SalesChannelDTO>
  listSalesChannels(filters?: FilterableSalesChannelProps, config?: FindConfig<SalesChannelDTO>, sharedContext?: Context): Promise<SalesChannelDTO[]>
  listAndCountSalesChannels(filters?: FilterableSalesChannelProps, config?: FindConfig<SalesChannelDTO>, sharedContext?: Context): Promise<[SalesChannelDTO[], number]>
}
```

### 4.1 DTOs

```ts
type SalesChannelDTO = {
  id: string
  name: string
  description: string | null
  is_disabled: boolean
  metadata: Record<string, unknown> | null
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}

type CreateSalesChannelDTO = {
  name: string
  description?: string
  is_disabled?: boolean
  metadata?: Record<string, unknown>
}
```

## 5. Module Architecture

```
@medusajs/sales-channel
├── src/
│   ├── models/
│   │   └── sales-channel.ts
│   ├── services/
│   │   └── sales-channel-module-service.ts
│   ├── migrations/
│   └── index.ts
```

## 6. Workflows

| Workflow                          | Steps                                |
|-----------------------------------|--------------------------------------|
| `createSalesChannelsWorkflow`     | `createSalesChannelsStep`            |
| `updateSalesChannelsWorkflow`     | `updateSalesChannelsStep`            |
| `deleteSalesChannelsWorkflow`     | `deleteSalesChannelsStep`            |
| `linkProductsToSalesChannelWorkflow` | `linkProductsToSalesChannelStep`  |

Hooks emitted: `salesChannelsCreated`, `salesChannelsUpdated`, `salesChannelsDeleted`.

## 7. Module Links

Links are defined at the application level (not inside the module) and connect:

```
ProductModule.Product ──< link >── SalesChannelModule.SalesChannel
StockLocationModule.StockLocation ──< link >── SalesChannelModule.SalesChannel
ApiKeyModule.ApiKey ──< link >── SalesChannelModule.SalesChannel
```

These produce pivot tables (e.g., `product_sales_channel`, `stock_location_sales_channel`) managed by the link infrastructure. Querying these links happens via the Remote Query layer, not direct module service calls.

## 8. Channel Resolution from API Requests

When a store request includes the `x-publishable-api-key` header:
1. Middleware resolves the API key record from the API Key module.
2. The module link between `ApiKey` and `SalesChannel` is queried.
3. The resolved `sales_channel_id` is attached to `req.publishableKeyScopes`.
4. Downstream handlers filter products and inventory by this channel.

If the header is absent, the default sales channel of the store is used.

## 9. Cart Integration

Cart creation accepts a `sales_channel_id`. This scopes:
- **Product availability**: Only products linked to the channel can be added.
- **Stock location priority**: Fulfillment routes through locations linked to the channel.
- **Order association**: The order inherits the cart's sales channel for reporting.

## 10. Error Handling

| Condition                             | Error Type                        |
|---------------------------------------|-----------------------------------|
| Sales channel not found               | `MedusaError.Types.NOT_FOUND`     |
| Adding product not linked to channel  | Validation in cart step           |
| Deleting active default channel       | `MedusaError.Types.NOT_ALLOWED`   |
