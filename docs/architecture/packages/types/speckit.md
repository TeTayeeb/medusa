# SpecKit — @medusajs/framework/types

**Package:** `@medusajs/framework/types`  
**Version:** 2.15.4  
**Document type:** Specification & Test Contracts  

---

## 1. Overview

This SpecKit defines the behavioural contracts for the `types` package. Because the package produces no runtime code, "behaviour" here means **TypeScript compiler behaviour** — what compiles, what errors, and what invariants must hold across package versions.

---

## 2. Type Contract Specifications

### SPEC-TYPES-01 — `HttpTypes` completeness

**Description:** Every Admin and Store API endpoint documented in the OpenAPI specification must have a corresponding type in `HttpTypes`.

**Verifiable by:**
- A build-time script that diffs the OpenAPI `paths` object against the `HttpTypes` namespace exports.
- CI fails if any endpoint is missing a request or response type.

**Acceptance criteria:**
- `HttpTypes.Admin{Resource}ListParams` exists for every list endpoint.
- `HttpTypes.Admin{Resource}Response` exists for every GET single endpoint.
- `HttpTypes.AdminCreate{Resource}` exists for every POST endpoint.
- Equivalent `HttpTypes.Store*` types exist for all storefront endpoints.

---

### SPEC-TYPES-02 — `Context` propagation safety

**Description:** The `Context` type must accept `undefined` as a value so that callers can pass `{}` as a default without TypeScript errors.

```typescript
// ✓ Must compile
function doWork(ctx: Context = {}) {}

// ✓ Must compile — all fields are optional
const ctx: Context = {
  transactionManager: undefined,
  enableNestedTransactions: true,
}

// ✗ Must not compile — unknown field
const badCtx: Context = { unknownField: "x" }
```

---

### SPEC-TYPES-03 — `BaseFilterable` composition

**Description:** `BaseFilterable<T>` must allow arbitrarily nested `$and` / `$or` combinations.

```typescript
type ProductFilter = BaseFilterable<{ id: string; status: string }> & {
  id?: string
  status?: string
}

// ✓ All of these must compile:
const f1: ProductFilter = { id: "prod_123" }
const f2: ProductFilter = { $and: [{ id: "prod_123" }, { status: "published" }] }
const f3: ProductFilter = { $or: [{ id: "a" }, { $and: [{ status: "draft" }] }] }
```

---

### SPEC-TYPES-04 — Module service interface completeness

**Description:** Every module service interface must extend `IModuleService` and declare all CRUD operations.

**Checklist per interface (`IProductModuleService`, `IOrderModuleService`, etc.):**

| Method pattern | Required |
|---|---|
| `retrieve{Entity}(id, config?, context?)` | ✓ |
| `list{Entities}(filters?, config?, context?)` | ✓ |
| `listAndCount{Entities}(filters?, config?, context?)` | ✓ |
| `create{Entities}(data[], context?)` | ✓ |
| `update{Entities}(data[], context?)` | ✓ |
| `delete{Entities}(ids[], context?)` | ✓ |
| `softDelete{Entities}(ids[], context?)` | ✓ |
| `restore{Entities}(ids[], context?)` | ✓ |

---

### SPEC-TYPES-05 — `FindConfig` field selection typing

**Description:** The `select` field of `FindConfig<T>` must accept strings corresponding to `keyof T` (dot-notation for relations).

```typescript
import type { FindConfig, ProductDTO } from "@medusajs/framework/types"

// ✓ Valid field names
const config: FindConfig<ProductDTO> = {
  select: ["id", "title", "variants.id"],
  take: 10,
}
```

---

### SPEC-TYPES-06 — No runtime exports

**Description:** Importing from `@medusajs/framework/types` must not add any bytes to a JavaScript bundle.

**Verification method:**
```bash
# After build, the bundle must not import types
node -e "const t = require('@medusajs/types'); console.log(Object.keys(t))"
# Must output: [] (empty — only type exports)
```

---

## 3. Breaking Change Rules

The following changes are **always breaking** and require a major version bump:

| Change | Reason |
|---|---|
| Removing an exported type or interface | Breaks all consumers that reference it |
| Renaming an exported type | Same as removal from consumer's perspective |
| Making an optional field required | Breaks object literals that omit the field |
| Narrowing a field's type (e.g., `string` → `"a" \| "b"`) | Breaks assignments of the wider type |
| Removing a method from a service interface | Breaks mock implementations |

The following changes are **always safe** (patch or minor):

| Change | Reason |
|---|---|
| Adding an optional field to a type | Structurally compatible |
| Adding a new type or interface | No existing code is affected |
| Adding an optional method to a service interface | Existing implementations still satisfy the interface |
| Widening a field's type (e.g., `"a"` → `string`) | Assignable subset is still valid |

---

## 4. Test Matrix

| Spec | Test Type | Location |
|---|---|---|
| SPEC-TYPES-01 | Build script | `scripts/check-http-types-coverage.ts` |
| SPEC-TYPES-02 | Type-level test (`@ts-expect-error`) | `src/__tests__/context.test-d.ts` |
| SPEC-TYPES-03 | Type-level test | `src/__tests__/filterable.test-d.ts` |
| SPEC-TYPES-04 | Structural check | `src/__tests__/service-interfaces.test-d.ts` |
| SPEC-TYPES-05 | Type-level test | `src/__tests__/find-config.test-d.ts` |
| SPEC-TYPES-06 | Bundle analysis | `scripts/check-types-bundle-size.sh` |

---

## 5. Glossary

| Term | Definition |
|---|---|
| **DTO** | Data Transfer Object — a plain, serialisable representation of an entity |
| **HttpTypes** | The TypeScript namespace containing all HTTP request/response types |
| **BaseFilterable** | A generic type adding `$and`/`$or` combinators to any filter type |
| **Context** | Ambient carrier for a database transaction and request metadata |
| **FindConfig** | Options object for querying: fields, relations, pagination, sort order |
