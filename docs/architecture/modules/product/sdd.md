# Software Design Document — Product Module

> Medusa v2.15.4

## 1. Module Architecture

```
packages/modules/product/src/
├── index.ts                    # Module entry, DI container wiring
├── joiner-config.ts            # Remote query join configuration
├── models/                     # MikroORM entity definitions (model DSL)
│   ├── product.ts
│   ├── product-variant.ts
│   ├── product-option.ts
│   ├── product-option-value.ts
│   ├── product-collection.ts
│   ├── product-category.ts
│   ├── product-tag.ts
│   ├── product-type.ts
│   ├── product-image.ts
│   └── product-variant-product-image.ts
├── repositories/
│   └── product.ts              # Custom repository extending base DAL
├── services/
│   ├── product-module-service.ts   # IProductModuleService implementation
│   ├── product-category.ts         # Category-specific queries (tree support)
│   └── index.ts
├── migrations/                 # MikroORM migration files
├── schema/                     # GraphQL schema fragments
├── types/                      # Internal DTOs (Update* input shapes)
└── utils/
    └── events.ts               # Event builder helpers
```

The architecture follows the Medusa **module pattern**: a single `ProductModuleService` implements `IProductModuleService` and extends `MedusaService<{...}>`, which provides auto-generated CRUD methods for every registered entity. Specialised operations (e.g. category tree traversal, handle auto-generation) are hand-coded on the service.

---

## 2. Data Model

### Entity-Relationship Overview

```
ProductType (1) ──── (*) Product (*) ──── (*) ProductTag
                          │
                   ProductCollection (1) ──── (*) Product
                          │
                   ProductCategory (M:M)
                          │
                   ProductOption (1:M)
                          │
                   ProductOptionValue (M:1) ── ProductVariant
                          │
                   ProductVariant (1:M)
                          │
              ProductVariantProductImage (pivot)
                          │
                   ProductImage
```

### Key Constraints
- `Product.handle` is **unique** (partial index where `deleted_at IS NULL`). Auto-derived from title via `kebabCase()` if not provided.
- `ProductCategory` is a **self-referential tree** via `parent_category_id`. The service computes `category_children` and ancestor paths.
- `ProductOptionValue` links variants to options; a variant must carry exactly one value per option defined on its parent product.
- All entities use **soft-delete** (`deleted_at` timestamp). Hard deletes are explicit.
- Cascade deletes: `Product → variants, options, images`; `ProductVariant → optionValues`.

---

## 3. Service Layer Design

### Class Hierarchy

```
MedusaService<EntityMap>          (framework/utils)
        └── ProductModuleService  (product module)
                ├── Auto-CRUD: list*, retrieve*, create*, update*, delete*, softDelete*, restore*
                ├── Custom: createProducts (handle generation, image reconciliation)
                ├── Custom: updateProducts (diff-based variant/option sync)
                └── ProductCategoryService  (injected sub-service for tree ops)
```

### Dependency Injection

All sub-services are injected via the `InjectedDependencies` container map at construction time:

```ts
type InjectedDependencies = {
  baseRepository: DAL.RepositoryService
  productRepository: ProductRepository        // custom repository
  productService: IMedusaInternalService      // CRUD for Product
  productVariantService: IMedusaInternalService
  productCategoryService: ProductCategoryService
  productCollectionService: IMedusaInternalService
  productTagService: IMedusaInternalService
  productTypeService: IMedusaInternalService
  productOptionService: IMedusaInternalService
  productOptionValueService: IMedusaInternalService
  productImageService: IMedusaInternalService
  [Modules.EVENT_BUS]?: IEventBusModuleService  // optional
}
```

### Transaction Management

- Public methods are decorated with `@InjectManager()` — they receive a fresh EntityManager.
- Protected inner methods (`*_`) use `@InjectTransactionManager()` to participate in an existing transaction.
- The `@MedusaContext()` decorator propagates the `SharedContext` (including `transactionManager`) through the call stack.

---

## 4. Repository Pattern

`ProductRepository` extends the base DAL repository and overrides `findAndCount` to support:
- Full-text search across `title`, `description`, `subtitle`, `variants.title` (searchable fields)
- Category sub-tree filtering (`category_id` with `include_descendants_tree`)
- Efficient join strategies for deeply nested relations

Standard entity repositories (`productVariantService`, etc.) use `ModulesSdkUtils.MedusaInternalService` which delegates to the MikroORM `EntityManager` directly.

---

## 5. Events Emitted

Events are built by helpers in `utils/events.ts` and emitted via `MessageAggregator` after transaction commit (decorated with `@EmitEvents()`).

| Event | Payload |
|---|---|
| `product.created` | `{ id: string }[]` |
| `product.updated` | `{ id: string }[]` |
| `product.deleted` | `{ id: string }[]` |
| `product-variant.created` | `{ id: string }[]` |
| `product-variant.updated` | `{ id: string }[]` |
| `product-variant.deleted` | `{ id: string }[]` |
| `product-category.created` | `{ id: string }[]` |
| `product-category.updated` | `{ id: string }[]` |
| `product-collection.created` | `{ id: string }[]` |

All events are published **after** the DB transaction commits, ensuring consumers see consistent state.

---

## 6. Error Handling

| Scenario | Error Type | Message Pattern |
|---|---|---|
| Product not found | `MedusaError.Types.NOT_FOUND` | `"Product with id: {id} was not found"` |
| Invalid handle | `MedusaError.Types.INVALID_DATA` | `"Product handle must be a valid slug"` |
| Duplicate handle | Database unique constraint → wrapped | `"Product with handle {handle} already exists"` |
| Invalid status value | `MedusaError.Types.INVALID_DATA` | Enum validation |
| Category cycle | `MedusaError.Types.INVALID_DATA` | Circular parent check |

All errors propagate through the workflow compensation system, triggering rollback of any partially committed state.
