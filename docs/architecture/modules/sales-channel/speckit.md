# Specification — Sales Channel Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                                         |
|-----------------|-----------------------------------------------|
| Module ID       | `Modules.SALES_CHANNEL`                       |
| Package         | `@medusajs/sales-channel`                     |
| Medusa Version  | 2.15.4                                        |
| Type            | Commerce Configuration                        |
| Database tables | `sales_channel`                               |
| API surface     | Admin (CRUD); Store (implicit via API key)    |

## 2. Functional Requirements

### FR-SC-01: Create Sales Channel
- **Given** a unique channel name
- **When** `POST /admin/sales-channels` is called
- **Then** a new SalesChannel is created and returned

### FR-SC-02: List Sales Channels
- **Given** an authenticated admin request
- **When** `GET /admin/sales-channels` is called with optional filters
- **Then** return paginated list of sales channels

### FR-SC-03: Update Sales Channel
- **Given** an existing sales channel ID
- **When** `POST /admin/sales-channels/:id` is called with updated fields
- **Then** the channel is updated; unchanged fields are preserved

### FR-SC-04: Disable Sales Channel
- **Given** an existing sales channel
- **When** `is_disabled: true` is set via update
- **Then** the channel is deactivated; API key requests resolving to this channel are rejected

### FR-SC-05: Delete Sales Channel
- **Given** a non-default sales channel
- **When** `DELETE /admin/sales-channels/:id` is called
- **Then** the channel is soft-deleted; product and location links are removed

### FR-SC-06: Link Products to Channel
- **Given** a sales channel ID and a list of product IDs
- **When** the link workflow runs (via Admin UI or API)
- **Then** those products become available on the channel (inserted into `product_sales_channel` link table)

### FR-SC-07: Channel Resolution via Publishable API Key
- **Given** a storefront request with `x-publishable-api-key` header
- **When** the middleware processes the request
- **Then** the key is resolved to a `sales_channel_id` via the `api_key ↔ sales_channel` link
- **If** the channel is disabled, the request returns 400 or falls back to default

### FR-SC-08: Default Channel Guarantee
- **Given** a store configuration
- **When** any request has no `sales_channel_id` context
- **Then** the store's `default_sales_channel_id` is used as fallback

### FR-SC-09: Cart Scoping
- **Given** a cart creation request with `sales_channel_id`
- **When** a product is added to the cart
- **Then** the product must be linked to that sales channel; otherwise, reject with 400

## 3. Non-Functional Requirements

| ID         | Requirement                            | Target                                              |
|------------|----------------------------------------|-----------------------------------------------------|
| NFR-SC-01  | Channel resolution latency             | < 50ms (cached API key → channel lookup)            |
| NFR-SC-02  | Product link table size                | Indexed; handles millions of product-channel pairs  |
| NFR-SC-03  | Default channel availability           | Always present; enforced at store configuration level |
| NFR-SC-04  | Disable takes effect immediately       | No cache delay; middleware checks `is_disabled` live |

## 4. Interface Specification

### POST `/admin/sales-channels`

| Attribute     | Value                                                                     |
|---------------|---------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                           |
| Body          | `{ name: string, description?: string, is_disabled?: boolean, metadata?: object }` |
| Response 200  | `{ sales_channel: SalesChannelDTO }`                                      |

### GET `/admin/sales-channels`

| Attribute     | Value                                                                         |
|---------------|-------------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                               |
| Query params  | `limit`, `offset`, `order`, `q`, `is_disabled`                               |
| Response 200  | `{ sales_channels: SalesChannelDTO[], count, limit, offset }`                |

## 5. Data Contracts

### SalesChannelDTO

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
```

### CreateSalesChannelDTO

```ts
type CreateSalesChannelDTO = {
  name: string
  description?: string
  is_disabled?: boolean   // default: false
  metadata?: Record<string, unknown>
}
```

### UpdateSalesChannelDTO

```ts
type UpdateSalesChannelDTO = {
  id: string
  name?: string
  description?: string
  is_disabled?: boolean
  metadata?: Record<string, unknown>
}
```

## 6. Module Link Contracts

| Link                              | Pivot Table                  | Direction           |
|-----------------------------------|------------------------------|---------------------|
| `product ↔ sales-channel`         | `product_sales_channel`      | Many-to-many        |
| `stock-location ↔ sales-channel`  | `stock_location_sales_channel` | Many-to-many      |
| `api-key ↔ sales-channel`         | `publishable_api_key_sales_channel` | Many-to-many  |

## 7. Edge Cases

| Case                                          | Expected Behaviour                                             |
|-----------------------------------------------|----------------------------------------------------------------|
| Delete a default sales channel                | Rejected with `MedusaError.Types.NOT_ALLOWED`                 |
| Publishable key resolves to disabled channel  | Request rejected (400) with descriptive error                 |
| Cart with no sales_channel_id                 | Falls back to store's default sales channel                   |
| Product not linked to requested channel       | Cannot be added to cart; returns 400                          |
| Duplicate channel names                       | Allowed (no uniqueness constraint on name)                    |
| Channel with no linked products               | Valid; shows empty product list in storefront                 |

## 8. Module Boundaries

| In Scope                                | Out of Scope                                          |
|-----------------------------------------|-------------------------------------------------------|
| Sales channel CRUD and registry         | Product catalogue management (Product module)         |
| `is_disabled` flag management           | Inventory management (Stock Location module)          |
| Module link participation (products, locations, keys) | API key creation (API Key module)      |
| Default channel fallback coordination   | Price list scoping per channel (Pricing module)       |

## 9. Acceptance Criteria Summary

- [ ] `POST /admin/sales-channels` creates a channel with `is_disabled: false` by default
- [ ] `GET /admin/sales-channels?is_disabled=false` excludes disabled channels
- [ ] `x-publishable-api-key` header resolves to correct `sales_channel_id`
- [ ] Adding product not linked to channel to a cart returns 400
- [ ] `DELETE` on the default sales channel returns an error
- [ ] Soft-deleted channels do not appear in default list queries
- [ ] Channel linked to multiple API keys; all resolve to same channel
