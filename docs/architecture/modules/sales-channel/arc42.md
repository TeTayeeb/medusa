# Architecture Documentation — Sales Channel Module (arc42)

## 1. Introduction and Goals

The Sales Channel module enables Medusa to power multiple distinct retail surfaces simultaneously. Rather than duplicating product catalogues or inventory, it uses a linking architecture to scope existing data to specific channels. This makes it the coordination hub for multi-channel commerce without embedding business logic from other modules.

**Quality Goals:**

| Priority | Quality Goal   | Description                                                                 |
|----------|----------------|-----------------------------------------------------------------------------|
| 1        | Modularity     | Channel scoping via links, not by duplicating product/inventory data        |
| 2        | Flexibility    | Any surface (web, mobile, POS, B2B) modelled as a sales channel             |
| 3        | Security       | Publishable API keys isolate channel access for storefronts                 |
| 4        | Simplicity     | The module itself is intentionally thin; complexity in links and workflows  |

## 2. Constraints

- A store must always have at least one active default sales channel.
- Product availability across channels is managed via module links, not by the Sales Channel module directly.
- The `x-publishable-api-key` header is the only mechanism for channel resolution in Store API requests.

## 3. Context and Scope

```
External:
  [Admin Browser]    ──CRUD──────► [Admin API /admin/sales-channels]
  [Storefront]       ──x-publishable-api-key header──► Channel resolved automatically
  [POS Terminal]     ──x-publishable-api-key header──► Different channel resolved

Internal:
  [Sales Channel Module] ──linked to──► [Product Module] (product availability)
  [Sales Channel Module] ──linked to──► [Stock Location Module] (inventory)
  [Sales Channel Module] ──linked to──► [API Key Module] (key resolution)
  [Cart Module]          ──reads────► [Sales Channel Module] (channel context)
```

## 4. Solution Strategy

| Challenge                                 | Strategy                                                         |
|-------------------------------------------|------------------------------------------------------------------|
| Product scoping without data duplication  | Module link table `product_sales_channel` as a pivot             |
| Storefront authentication/channel binding | Publishable API keys linked to channels; resolved in middleware  |
| Inventory routing per channel             | `stock_location_sales_channel` link used at fulfillment step     |
| Default channel guarantee                 | Store module maintains `default_sales_channel_id`                |

## 5. Building Block View

```
Sales Channel Module
├── HTTP Layer
│   └── Admin Routes (GET/POST/DELETE /admin/sales-channels)
│
├── Workflow Layer
│   ├── createSalesChannelsWorkflow
│   ├── updateSalesChannelsWorkflow
│   ├── deleteSalesChannelsWorkflow
│   └── linkProductsToSalesChannelWorkflow
│
├── Service Layer
│   └── SalesChannelModuleService
│       ├── createSalesChannels / updateSalesChannels / deleteSalesChannels
│       └── listSalesChannels / retrieveSalesChannel
│
├── Domain Model
│   └── SalesChannel
│
└── Module Link Tables (external to module)
    ├── product_sales_channel
    ├── stock_location_sales_channel
    └── api_key_sales_channel
```

## 6. Runtime View

**Scenario A: Storefront request with publishable key**

```
GET /store/products
  x-publishable-api-key: pk_01HXXX
  → Middleware: resolves ApiKey from API Key module
  → Follows api_key ↔ sales_channel link
  → Sets req.publishableKeyScopes = { sales_channel_id: "sc_01HXXX" }
  → Product list handler: applies sales_channel_id filter via remote query
  → Only products linked to sc_01HXXX returned
```

**Scenario B: Add product to sales channel (Admin)**

```
POST /admin/sales-channels/:id/products { add: ["prod_01HXXX"] }
  → Route handler calls linkProductsToSalesChannelWorkflow.run()
  → createRemoteLinkStep: inserts into product_sales_channel pivot
  → Product now available on the channel
```

## 7. Deployment View

Single process, same PostgreSQL database. The module itself uses one table (`sales_channel`). Link tables are created by the Medusa link infrastructure in the same database.

## 8. Cross-Cutting Concerns

| Concern         | Approach                                                                   |
|-----------------|----------------------------------------------------------------------------|
| Authentication  | Admin API: JWT required. No public Store endpoint for channel management   |
| Publishable keys| Middleware validation; keys are hashed and resolved via API Key module     |
| Transactions    | `@InjectTransactionManager()` for write operations                         |
| Events          | `salesChannelsCreated`, `salesChannelsUpdated`, `salesChannelsDeleted` hooks |

## 9. Design Decisions

| ID  | Decision                                 | Rationale                                                              |
|-----|------------------------------------------|------------------------------------------------------------------------|
| D1  | No Store write API                        | Channels are merchant-configured; customers select, not configure      |
| D2  | Scoping via module links, not FK columns  | Avoids coupling Product and SalesChannel modules at the schema level   |
| D3  | Single entity (no sub-entities)           | Channel config is flat; provider associations handled externally       |
| D4  | `is_disabled` flag instead of delete     | Allows temporary channel suspension without destroying link data       |

## 10. Risks and Technical Debt

| Risk                                      | Mitigation                                                  |
|-------------------------------------------|-------------------------------------------------------------|
| Large product catalogues with many links  | Indexed pivot table; link queries paginated                 |
| API key leak exposing a channel           | Keys are hashed; publishable keys have limited scope        |
| Default channel deletion                  | Guard in `deleteSalesChannelsWorkflow` prevents orphan state |
