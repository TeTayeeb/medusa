# commerce-catalog — Contracts

## Public Interface

```
ports/commerce-catalog.contract.ts
```

### `CommerceCatalogContract`

```typescript
interface CommerceCatalogContract {
  healthCheck(): Promise<{ module: "commerce-catalog"; status: "ok" }>
}
```

## API Surface

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/store/products` | Public | List products with catalog filters |
| GET | `/store/products/:id` | Public | Retrieve a product with full catalog data |
| GET | `/admin/products` | Admin | Admin product list with extended metadata |
| POST | `/admin/products` | Admin | Create product with catalog attributes |

## Query Convention

All data retrieval uses `query.graph()` (Medusa Query API).

```typescript
const { data } = await query.graph({
  entity: "product",
  fields: ["id", "title", "variants.prices", "variants.inventory_quantity"],
  filters: { status: "published" },
})
```

Never call `productModuleService.listProducts()` directly from a route.

## Contract Stability Rules

Consumers depend only on `CommerceCatalogContract`.
Run `yarn test:backend:contracts` after any change.
