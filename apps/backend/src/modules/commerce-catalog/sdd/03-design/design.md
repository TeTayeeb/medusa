# commerce-catalog — Design

## Architecture

Port/Adapter pattern:

```
ports/commerce-catalog.contract.ts   ← stable interface
adapters/medusa/                     ← Medusa product/pricing implementation
__tests__/contract.spec.ts           ← type-checked against contract
```

## Port Interface

See `ports/commerce-catalog.contract.ts`.

## Adapter

`CommerceCatalogMedusaAdapter` uses:
- `@medusajs/product` module APIs
- `@medusajs/pricing` module for price lists
- Feature flag: `FEATURE_COMMERCE_CATALOG_V2`

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/store/products` | List products with catalog filters |
| GET | `/store/products/:id` | Retrieve a product with full catalog data |
| GET | `/admin/products` | Admin product list with extended metadata |
| POST | `/admin/products` | Create product with catalog attributes |

## Upgrade Safety

Queries use `query.graph()` — not direct module service calls from routes.
