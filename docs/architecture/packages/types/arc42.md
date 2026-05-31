# arc42 Architecture Documentation — @medusajs/framework/types

**Package:** `@medusajs/framework/types`  
**Version:** 2.15.4  
**Template:** arc42 v8  

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/framework/types` exists to provide a single, authoritative TypeScript contract for the entire Medusa platform. Every package in the monorepo — HTTP routes, module services, workflow steps, and the JS SDK — must share a common vocabulary. Without this package, each layer would define its own shapes and drift would be inevitable.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Consistency** | A `ProductDTO` returned by a module service is structurally identical to the product object in an HTTP response. |
| 2 | **Zero runtime cost** | Adding a new import from `types` must not add bytes to any production bundle. |
| 3 | **Discoverability** | A developer can find the type for any API endpoint or module method within seconds. |
| 4 | **Backward compatibility** | Existing code must not break when the package receives a patch or minor version bump. |

---

## 2. Architecture Constraints

- **No runtime code** — TypeScript `type` and `interface` only; no `const`, `class`, or `function` exports that produce JavaScript.
- **No circular imports** — `types` must not import from any other Medusa package.
- **OpenAPI alignment** — `HttpTypes` must stay in sync with the published OpenAPI spec; divergence is a release blocker.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────┐
│                  Medusa Monorepo                    │
│                                                     │
│   ┌─────────┐   ┌──────────┐   ┌────────────────┐  │
│   │  utils  │   │modules-sdk│  │  js-sdk        │  │
│   └────┬────┘   └────┬─────┘  └───────┬────────┘  │
│        │             │                │            │
│        └─────────────┴────────────────┘            │
│                       │                            │
│              ┌─────────▼────────┐                  │
│              │      types       │  ← this package  │
│              └──────────────────┘                  │
└─────────────────────────────────────────────────────┘
```

All arrows point *into* `types`. It has no outbound dependencies.

---

## 4. Solution Strategy

### 4.1 Namespace partitioning

Types are grouped by domain and layer:

| Namespace / folder | Layer | Contents |
|---|---|---|
| `http/admin/` | API | Admin endpoint request/response types |
| `http/store/` | API | Store endpoint request/response types |
| `product/`, `order/`, … | Module | Service interfaces + DTOs |
| `dal/` | Infrastructure | Database query / filter types |
| `shared-context.d.ts` | Cross-cutting | Transaction context carrier |
| `dml/` | Infrastructure | Data Modelling Layer types |
| `workflow/` | Workflow | Step input/output types |

### 4.2 Barrel exports

Each domain folder has an `index.d.ts` that re-exports all its types. The root `index.d.ts` re-exports all domain barrels. Consumers can import broadly (`from "@medusajs/framework/types"`) or narrowly (`from "@medusajs/framework/types/product"`) — both are tree-shaken identically since all exports are type-only.

---

## 5. Building Block View

```
types/
├── http/
│   ├── admin/      (Admin* types per resource)
│   └── store/      (Store* types per resource)
├── product/
│   ├── common.d.ts     ProductDTO, ProductVariantDTO, …
│   └── service.d.ts    IProductModuleService
├── order/
│   ├── common.d.ts
│   └── service.d.ts    IOrderModuleService
├── dal/
│   ├── index.d.ts      BaseFilterable, FindConfig, OptionsQuery
│   └── utils.d.ts      FilterQuery, OperatorMap
├── shared-context.d.ts Context, IMessageAggregator
└── index.d.ts          Root barrel
```

---

## 6. Runtime View

There is no runtime behaviour. The package is erased by the TypeScript compiler. The "runtime view" of this package is the TypeScript Language Server providing autocomplete and type-checking in editors.

---

## 7. Deployment View

`types` is published to npm as `@medusajs/types`. The framework re-exports it as `@medusajs/framework/types` via package export maps. Consumers never need to install `@medusajs/types` directly.

---

## 8. Cross-Cutting Concepts

### 8.1 Wrapping convention for HTTP responses

All HTTP responses wrap their payload:

```typescript
// ✓  { product: ProductDTO }
// ✗  ProductDTO  (bare)
```

This allows future addition of metadata fields without a breaking change.

### 8.2 `$and` / `$or` filter combinators

All filterable types extend `BaseFilterable<T>`, which adds `$and` and `$or`. This enables arbitrarily nested boolean filter trees that the query builder translates to SQL `AND` / `OR`.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | Type-only package, no JS emission | Zero runtime overhead; TypeScript compiler enforces contracts. |
| AD-02 | `HttpTypes` namespace (not separate modules) | Keeps all API types importable from one path. |
| AD-03 | `Context<TManager>` generic | Allows the same Context type to work with MikroORM and custom ORMs. |

---

## 10. Quality Requirements

| Scenario | Response Measure |
|---|---|
| Developer adds optional field to `ProductDTO` | Existing code continues to compile without changes. |
| Developer renames a required field | TypeScript errors appear in all consumers; caught in CI before merge. |
| New module added | Add domain folder + `service.d.ts` + re-export in root barrel. |

---

## 11. Risks and Technical Debt

- **Manual sync with OpenAPI** — the `HttpTypes` sub-namespace is partially hand-authored. A generator would eliminate drift risk.
- **No runtime validation** — types alone don't prevent bad data entering the system; `zod` co-generation is a future consideration.
