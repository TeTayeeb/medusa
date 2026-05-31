# Product Module — `@medusajs/product`

> Medusa v2.15.4 · Module Reference

## Purpose & Domain Responsibility

The Product Module owns the entire **product catalog** domain. It is the single source of truth for all merchandise data: products, their variants, organisational taxonomy (categories, collections, types, tags), purchasing options, imagery, and SEO handles. No other module writes product data; every other module (Cart, Order, Pricing, Inventory) references products by ID.

---

## Key Entities

| Entity | Prefix | Description |
|---|---|---|
| `Product` | `prod_` | Root catalog item. Carries title, handle, status, dimensions, metadata. |
| `ProductVariant` | `variant_` | A purchasable SKU of a product (e.g. "Blue / XL"). Holds SKU, barcode, weight overrides, manage-inventory flag. |
| `ProductOption` | `opt_` | A configurable dimension of a product (e.g. "Color", "Size"). |
| `ProductOptionValue` | `optval_` | A concrete value for an option on a specific variant (e.g. "Blue"). |
| `ProductCollection` | `pcol_` | A curated group of products (e.g. "Summer 2025"). |
| `ProductCategory` | `pcat_` | Hierarchical taxonomy node. Supports unlimited nesting via `parent_category_id`. |
| `ProductTag` | `ptag_` | Free-form tag for cross-cutting classification. Many-to-many with products. |
| `ProductType` | `ptyp_` | A single-value type label for a product (e.g. "Apparel"). |
| `ProductImage` | `img_` | Image URL attached to a product or variant. |

---

## Key Service Methods

```ts
// Read
listProducts(filters, config, context): Promise<ProductDTO[]>
retrieveProduct(id, config, context): Promise<ProductDTO>
listProductVariants(filters, config, context): Promise<ProductVariantDTO[]>
listProductCategories(filters, config, context): Promise<ProductCategoryDTO[]>
listProductCollections(filters, config, context): Promise<ProductCollectionDTO[]>
listProductTags(filters, config, context): Promise<ProductTagDTO[]>
listProductTypes(filters, config, context): Promise<ProductTypeDTO[]>

// Write
createProducts(data[], context): Promise<ProductDTO[]>
updateProducts(selector, data, context): Promise<ProductDTO[]>
deleteProducts(ids[], context): Promise<void>

createProductVariants(data[], context): Promise<ProductVariantDTO[]>
updateProductVariants(selector, data, context): Promise<ProductVariantDTO[]>
deleteProductVariants(ids[], context): Promise<void>

createProductOptions(data[], context): Promise<ProductOptionDTO[]>
updateProductOptions(selector, data, context): Promise<ProductOptionDTO[]>

createProductCategories(data[], context): Promise<ProductCategoryDTO[]>
updateProductCategories(selector, data, context): Promise<ProductCategoryDTO[]>
deleteProductCategories(ids[], context): Promise<void>

createProductCollections(data[], context): Promise<ProductCollectionDTO[]>
upsertProductTags(data[], context): Promise<ProductTagDTO[]>
```

---

## Module Dependencies

| Dependency | Direction | Reason |
|---|---|---|
| `@medusajs/event-bus` | Optional, outbound | Emits catalog-change events for downstream subscribers |
| `@medusajs/framework/utils` | Required | `MedusaService`, decorators, `ProductStatus` enum |
| `@medusajs/framework/types` | Required | DTO types and interface contracts |

The Product Module is deliberately **standalone** — it holds no hard dependency on Cart, Order, Pricing, or Inventory. Those modules reference product / variant IDs as plain strings.

---

## Key Workflows

| Workflow | Package | Description |
|---|---|---|
| `createProductWorkflow` | `@medusajs/core-flows` | Validates & persists a new product with variants, options, images, tags, categories |
| `updateProductWorkflow` | `@medusajs/core-flows` | Applies partial updates; reconciles variant additions/removals |
| `deleteProductWorkflow` | `@medusajs/core-flows` | Soft-deletes product, cascades to variants, options, images |
| `batchProductsWorkflow` | `@medusajs/core-flows` | Bulk create / update / delete in a single transaction |
| `importProductsWorkflow` | `@medusajs/core-flows` | CSV-based batch import via file service |
| `exportProductsWorkflow` | `@medusajs/core-flows` | CSV export with progress notification |
| `createProductVariantsWorkflow` | `@medusajs/core-flows` | Adds variants to existing products |
| `deleteProductVariantsWorkflow` | `@medusajs/core-flows` | Removes variants; cascades through pricing & inventory |

---

## Admin API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/admin/products` | List products with filters, sorting, pagination |
| `POST` | `/admin/products` | Create product |
| `GET` | `/admin/products/:id` | Retrieve product |
| `POST` | `/admin/products/:id` | Update product |
| `DELETE` | `/admin/products/:id` | Delete product |
| `POST` | `/admin/products/:id/variants` | Create variant |
| `POST` | `/admin/products/:id/options` | Create option |
| `GET` | `/admin/product-variants` | List all variants across products |
| `GET` | `/admin/collections` | List collections |
| `POST` | `/admin/collections` | Create collection |
| `GET` | `/admin/categories` | List categories (tree) |
| `POST` | `/admin/categories` | Create category |
| `GET` | `/admin/product-tags` | List tags |
| `GET` | `/admin/product-types` | List types |

---

## Configuration Options

```ts
// medusa-config.ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/product",
      options: {
        // Pass a custom database connection (defaults to shared DB)
        database: { clientUrl: "postgresql://..." },
      },
    },
  ],
})
```

The module also participates in the **Module Link** system — no extra configuration is needed for cross-module queries via `remoteQuery`.

---

## Extension Points

| Extension | How |
|---|---|
| Custom product fields | Add `additional_data` via API routes; persist with `createProductWorkflow` hooks |
| Event hooks | Subscribe to `product.created`, `product.updated`, `product.deleted` via Event Bus |
| Workflow hooks | `productsCreated`, `productsUpdated`, `productsDeleted` hooks on each workflow |
| Custom service | Extend `ProductModuleService` and re-register in DI container |
| Custom validation | Inject middleware on `/admin/products` routes |
