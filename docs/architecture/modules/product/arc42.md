# arc42 Architecture Documentation — Product Module

> Medusa v2.15.4 | Focused sections: 1, 3, 5, 6, 8, 9

---

## Section 1 — Introduction and Goals

### 1.1 Purpose

The Product Module provides a **bounded context** for the commerce product catalog. It is the authoritative store for all product and variant data that other modules consume by reference (ID). Its primary stakeholders are:

- **Merchants** — manage catalog via Admin API
- **Storefront developers** — read catalog via Store API / `remoteQuery`
- **Downstream modules** — Cart, Order, Pricing, Inventory reference `product_id` / `variant_id`

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Correctness** | Product handle uniqueness must be enforced; variant-option consistency must hold |
| 2 | **Extensibility** | Merchants must be able to add custom fields via `additional_data` without module changes |
| 3 | **Performance** | `listProducts` with nested variants/categories must return < 200 ms at p95 for catalogs up to 100k products |
| 4 | **Isolation** | Product module must function independently; no circular module dependencies |

---

## Section 3 — Building Block View

### Level 1 — Context

```
┌──────────────────────────────────────────────┐
│              Medusa Application               │
│  ┌──────────────┐   ┌───────────────────────┐│
│  │  Admin API   │──▶│  ProductModuleService  ││
│  │  Store API   │   │  (IProductModuleService)│
│  └──────────────┘   └───────────┬───────────┘│
│                                 │            │
│                    ┌────────────▼───────────┐│
│                    │  MikroORM EntityManager ││
│                    │  (PostgreSQL)           ││
│                    └────────────────────────┘│
│  ┌──────────────┐                            │
│  │  Event Bus   │◀── product.created/updated │
│  └──────────────┘                            │
└──────────────────────────────────────────────┘
```

### Level 2 — Internal Decomposition

| Component | Responsibility |
|---|---|
| `ProductModuleService` | Orchestrates all catalog CRUD; handle gen; image reconciliation |
| `ProductCategoryService` | Tree traversal; ancestor path computation |
| `ProductRepository` | Custom query builder for full-text + category-tree search |
| `eventBuilders` | Constructs typed event payloads from entity arrays |
| `joinerConfig` | Declares join linkages for `remoteQuery` federation |

---

## Section 5 — Building Block: Runtime View

### Scenario: Create Product with Variants

```
API Handler → createProductWorkflow.run(input)
    │
    ├─ [step] validateProductStep          # validate handle, status, options
    ├─ [step] createProductsStep
    │      └─ ProductModuleService.createProducts()
    │              ├─ generate handle (kebabCase title)
    │              ├─ upsert ProductType / ProductTags
    │              ├─ create Product
    │              ├─ create ProductOptions + OptionValues
    │              ├─ create ProductVariants (linked to option values)
    │              └─ create ProductImages
    ├─ [step] createPricingStep            # Pricing module, separate transaction
    ├─ [step] attachInventoryItems         # Inventory module
    └─ [hook] productsCreated              # extensibility hook
```

All steps within a workflow run inside a **distributed saga**. If any step throws, compensation functions run in reverse order, calling `deleteProducts`, `deletePrices`, etc.

---

## Section 6 — Runtime View: List Products with Category Filter

```
remoteQuery({
  entryPoint: "product",
  variables: { filters: { category_id: ["pcat_01"] }, include_descendants_tree: true },
  fields: ["id", "title", "variants.*", "categories.*"]
})
    │
    ├─ ProductModuleService.listProducts(filters, config)
    │      └─ ProductRepository.findAndCount()
    │              ├─ resolve category descendant IDs
    │              ├─ join variants, options, option_values, images, tags
    │              └─ apply soft-delete filter
    └─ return hydrated ProductDTO[]
```

---

## Section 8 — Crosscutting Concepts

### Transactions

Every public service method is wrapped with `@InjectManager()`. The EntityManager is passed through `SharedContext`. Nested calls use `@InjectTransactionManager()` to reuse the same connection. Workflow steps each run in their own transaction; cross-step consistency is managed by workflow compensations.

### Soft Delete

All entities carry `created_at`, `updated_at`, `deleted_at` timestamps managed by MikroORM lifecycle hooks. All queries automatically filter `WHERE deleted_at IS NULL`. Restore operations set `deleted_at = NULL`.

### Handle Generation

If a product is created without an explicit `handle`, the service derives one from `title` using `kebabCase()`. Collision is resolved by appending a numeric suffix. Handles must pass `isValidHandle()` validation (alphanumeric + hyphens only).

### Event Aggregation

The `MessageAggregator` collects events during a transaction and flushes them after commit via `@EmitEvents()`. This prevents half-emitted event storms if a transaction partially fails.

---

## Section 9 — Architecture Decisions

### ADR-PROD-001: Soft Delete Over Hard Delete

**Decision**: All product deletes are soft (set `deleted_at`), not hard.  
**Rationale**: Orders and Carts reference `variant_id`; hard-deleting a variant would break historical order data.  
**Consequence**: Periodic purge jobs are needed to reclaim DB space.

### ADR-PROD-002: Handle as Stable URL Key

**Decision**: `Product.handle` is a unique, URL-safe slug that functions as a human-readable stable key.  
**Rationale**: SEO requires predictable URLs that survive title changes.  
**Consequence**: `handle` updates require explicit opt-in to avoid unintended URL changes.

### ADR-PROD-003: Option-Value Consistency at Write Time

**Decision**: The service validates that every variant carries exactly one `ProductOptionValue` per `ProductOption` defined on the product at create time.  
**Rationale**: Inconsistent option values corrupt the variant matrix in storefronts.  
**Consequence**: Bulk variant imports must always include full option value data.

### ADR-PROD-004: Separate ProductImage Entity

**Decision**: Images are stored as a dedicated entity rather than a JSON array.  
**Rationale**: Enables individual image ordering, metadata, and deletion without rewriting the entire array.  
**Consequence**: Requires a join when fetching product images; slightly higher query complexity.
