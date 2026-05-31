# @medusajs/framework/types

> Version: 2.15.4 · Package path: `packages/core/types`

The **types** package is the single source of truth for every TypeScript contract in the Medusa monorepo. No runtime code lives here — only type declarations, interfaces, and enumerations that every other package imports. This strict discipline ensures that the shape of data is decided in one place and propagated everywhere via the TypeScript compiler.

---

## Overview

```
@medusajs/framework/types
├── HTTP types          (HttpTypes.Admin*, HttpTypes.Store*)
├── Module service interfaces  (IProductModuleService, IOrderModuleService …)
├── DTO types           (ProductDTO, OrderDTO, CartDTO …)
├── Shared context      (Context / MedusaContext)
├── DAL types           (BaseFilterable, OptionsQuery, FindConfig …)
├── Module config types (ModuleDefinition, ModuleExports …)
├── Event types         (ProductEvents, OrderEvents …)
└── Workflow I/O types  (WorkflowData, WorkflowResponse …)
```

---

## Installation

```bash
# The types package is re-exported from the framework bundle
pnpm add @medusajs/framework
```

Import from the types sub-path:

```typescript
import type {
  Context,
  ProductDTO,
  IProductModuleService,
  HttpTypes,
} from "@medusajs/framework/types"
```

---

## Key Namespaces

### `HttpTypes` — API request/response contracts

Every Admin and Store API endpoint has a matching pair of request and response types grouped under the `HttpTypes` namespace.

```typescript
import type { HttpTypes } from "@medusajs/framework/types"

// Admin: list products request query
const query: HttpTypes.AdminProductListParams = {
  limit: 20,
  offset: 0,
  title: "shirt",
}

// Admin: single product response
const response: HttpTypes.AdminProductResponse = {
  product: {
    id: "prod_01HX...",
    title: "Classic Shirt",
    variants: [],
    // …
  },
}

// Store: cart creation
const createCartBody: HttpTypes.StoreCreateCart = {
  region_id: "reg_01HX...",
  currency_code: "usd",
  items: [{ variant_id: "variant_01HX...", quantity: 2 }],
}
```

### `Context` — shared transaction context

Passed as the last argument to every module service method so that a single database transaction can span multiple service calls.

```typescript
import type { Context } from "@medusajs/framework/types"

async function doWork(sharedContext: Context = {}) {
  await productModule.createProducts([{ title: "New" }], sharedContext)
  await inventoryModule.createInventoryItems([{ sku: "NEW-1" }], sharedContext)
  // Both run inside the same transaction
}
```

### Module service interfaces — `IProductModuleService`

```typescript
import type {
  IProductModuleService,
  ProductDTO,
  FilterableProductProps,
  FindConfig,
} from "@medusajs/framework/types"

// Resolve via container in a workflow step
const productService: IProductModuleService =
  container.resolve("productModuleService")

const products: ProductDTO[] = await productService.listProducts(
  { status: "published" } satisfies FilterableProductProps,
  { select: ["id", "title", "variants.id"], take: 10 } satisfies FindConfig<ProductDTO>
)
```

### DAL types — `BaseFilterable`, `FindConfig`

```typescript
import type { BaseFilterable, FindConfig } from "@medusajs/framework/types"

type MyFilter = BaseFilterable<{ id: string; status: string }> & {
  id?: string
  status?: string
}

const config: FindConfig<{ id: string; title: string }> = {
  select: ["id", "title"],
  relations: ["variants"],
  take: 50,
  skip: 0,
  order: { title: "ASC" },
}
```

---

## Directory Structure (dist)

```
dist/
├── address/              Cart/order address types
├── admin/                Admin-specific types
├── auth/                 Authentication types
├── cart/                 Cart entity types
├── common/               Shared utility types (FindConfig, SoftDelete…)
├── customer/             Customer types
├── dal/                  Data Access Layer types
├── dml/                  Data Modeling Layer types
├── event-bus/            Event message types
├── http/                 HttpTypes namespace (Admin + Store)
├── order/                Order entity types
├── payment/              Payment types
├── product/              Product entity + service interface
├── shared-context.d.ts   Context type definition
├── workflow/             Workflow I/O types
└── index.d.ts            Root barrel export
```

---

## Philosophy

- **Pure types, zero runtime** — tree-shaken away entirely in production JS bundles.
- **Versioned contracts** — a semver bump in this package signals breaking API changes.
- **Dual use** — the same type file describes both what an HTTP route returns _and_ what the underlying module service method returns, ensuring end-to-end consistency.
- **Strict null checks** — all types are authored with `strictNullChecks: true`; nothing is accidentally `any`.

---

## See Also

- [`@medusajs/framework/utils`](../utils/README.md) — runtime helpers that consume these types
- [`@medusajs/modules-sdk`](../modules-sdk/README.md) — module authoring utilities
- [Medusa API Reference](https://docs.medusajs.com/api/admin)
