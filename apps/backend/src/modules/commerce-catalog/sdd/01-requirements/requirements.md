# commerce-catalog — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| CAT-01 | The module MUST expose storefront product listing with catalog filters (category, collection, tags, price range). |
| CAT-02 | The module MUST expose a storefront product detail endpoint with full catalog data (variants, prices, images). |
| CAT-03 | The module MUST expose admin product listing with extended metadata. |
| CAT-04 | The module MUST allow creating products with custom catalog attributes via admin API. |
| CAT-05 | All product queries MUST use `query.graph()` — never direct module service calls from routes. |
| CAT-06 | The module MUST be togglable via `FEATURE_COMMERCE_CATALOG_V2`. |
| CAT-07 | The module MUST expose a health check returning `{ module: "commerce-catalog", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| CAT-NF-01 | Product list queries MUST return within 200 ms for catalogs up to 10,000 products (with Index Module). |
| CAT-NF-02 | Prices MUST always be included in storefront responses via `fields: ["variants.prices"]`. |
| CAT-NF-03 | The module MUST NOT import from Medusa private paths. |

## Out of Scope

- Inventory stock levels (handled natively by Medusa's inventory module)
- Payment and cart logic (handled by `checkout-payment`)
- Order management (handled by `order-fulfillment`)
