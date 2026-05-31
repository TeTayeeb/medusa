# Software Design Document — @medusajs/framework/types

**Package:** `@medusajs/framework/types`  
**Version:** 2.15.4  
**Status:** Stable  
**Owner:** Medusa Core Team

---

## 1. Purpose and Scope

This document describes the design of the `types` package, which owns all TypeScript contract definitions for the Medusa v2 platform. The package produces no JavaScript runtime artefacts — every export is erased by the TypeScript compiler. Its sole purpose is to express the shape of data flowing between HTTP clients, API routes, workflow steps, and module services.

---

## 2. Design Goals

| Goal | Rationale |
|---|---|
| **Single source of truth** | Any type change propagates to all consumers via the compiler, eliminating drift between layers. |
| **Zero runtime overhead** | Because no JS is emitted, including this package has no bundle-size cost. |
| **End-to-end consistency** | The same `ProductDTO` type used in a module service method is what the HTTP response type references — one change fixes both layers. |
| **Progressive disclosure** | Types are organised by domain. Consumers import only what they need. |

---

## 3. Namespace Design

### 3.1 `HttpTypes`

The `HttpTypes` namespace is the canonical description of the REST API surface. It is generated from the same source as the OpenAPI spec.

```
HttpTypes
├── Admin*ListParams      — query string for list endpoints (filters + pagination)
├── Admin*Response        — single-entity response wrapper
├── Admin*ListResponse    — paginated list response wrapper
├── AdminCreate*          — request body for create endpoints
├── AdminUpdate*          — request body for update endpoints
├── Store*                — same pattern for storefront endpoints
```

Design decision: wrapping every response in `{ product: ProductDTO }` rather than returning the DTO directly matches REST conventions and allows adding meta-fields (e.g., `{ product: ..., links: ... }`) in a non-breaking way.

### 3.2 Module service interfaces (`IProductModuleService`, etc.)

Each module exposes a service interface that extends `IModuleService`. The interface declares every public method with full input/output types. This allows:

- **Dependency inversion** — application code depends on the interface, not the concrete class.
- **Test mocking** — tests can inject a typed mock without importing the real service.
- **Cross-module type checking** — when a workflow step resolves a service from the DI container, the resolved type is the interface.

### 3.3 DTO types

DTOs (Data Transfer Objects) are plain-object types that describe entities as returned from module services. They are deliberately separate from the ORM entity classes:

- ORM entities carry MikroORM decorators and lifecycle hooks.
- DTOs are serialisable, dependency-free, and safe to pass across service boundaries.

### 3.4 `Context` / `MedusaContext`

```typescript
type Context<TManager = unknown> = {
  transactionManager?: TManager
  manager?: TManager
  isolationLevel?: string
  enableNestedTransactions?: boolean
  eventGroupId?: string
  transactionId?: string
  // … request metadata fields
}
```

The context acts as an ambient carrier for the current database transaction. Passing it as the last argument of every module service method — rather than using AsyncLocalStorage — makes the transaction flow explicit and testable.

### 3.5 DAL types

`BaseFilterable<T>` provides `$and` / `$or` combinators, mirroring MongoDB-style query composition but translated to SQL by the MikroORM query builder.

`FindConfig<T>` encodes `select`, `relations`, `take`, `skip`, and `order` in one object, matching MikroORM's `FindOptions` but typed to a specific entity shape.

---

## 4. Dependency Rules

```
types  ←  (no runtime dependencies)
       ←  consumed by: utils, modules-sdk, js-sdk, all modules, medusa core
```

Types may not import from any other Medusa package. This prevents circular dependencies.

---

## 5. Versioning Strategy

Type changes follow semantic versioning:

- **Patch** — adding optional properties to existing types.
- **Minor** — adding new type aliases, interfaces, or namespaces.
- **Major** — removing or renaming existing exports, or making optional fields required.

Because TypeScript's structural typing means adding required fields is always breaking, strict backward compatibility is enforced by the PR review process.

---

## 6. Type Generation

Several sub-namespaces under `HttpTypes` are generated from the OpenAPI specification during the build step. The generator is located in `scripts/generate-http-types`. Manual edits to generated files are prohibited; changes must be made to the OpenAPI spec instead.

Module service interface methods and DTO types are authored by hand alongside the module they describe, then re-exported from `types/src/product/`, `types/src/order/`, etc.

---

## 7. Testing

Because there is no runtime code, the types package is validated by:

1. **TypeScript compilation** — `tsc --noEmit` across the entire monorepo catches type errors.
2. **Type-level tests** — selected files use `@ts-expect-error` comments to assert that invalid assignments are rejected.
3. **Integration tests** — HTTP integration tests implicitly validate that response shapes match the declared `HttpTypes` by deserialising API responses into the typed variables.

---

## 8. Open Questions / Future Work

- Investigate generating DTO types from DML model definitions to eliminate manual drift.
- Evaluate shipping a JSON Schema artifact alongside TypeScript types for runtime validation use cases.
- Add `zod` schema co-generation for end-to-end type + runtime safety on API inputs.
