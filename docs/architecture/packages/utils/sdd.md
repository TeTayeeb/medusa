# Software Design Document — @medusajs/framework/utils

**Package:** `@medusajs/framework/utils`  
**Version:** 2.15.4  
**Status:** Stable  
**Owner:** Medusa Core Team

---

## 1. Purpose and Scope

The `utils` package is the shared runtime library for the Medusa v2 platform. It exports:

- The `MedusaService` factory that generates CRUD base classes from MikroORM entity definitions.
- A set of TypeScript decorators that apply cross-cutting concerns (transaction management, event emission, context injection).
- `MedusaError`, a typed exception hierarchy that maps to HTTP status codes.
- Identity generation, input validators, feature flags, and MikroORM connection utilities.

---

## 2. Design Goals

| Goal | Decision |
|---|---|
| **DRY module services** | `MedusaService` generates list/retrieve/create/update/delete for every entity automatically. |
| **Explicit transactions** | Decorators make it impossible to accidentally call a write method outside a transaction. |
| **Typed errors** | `MedusaError.Types` prevents magic strings and enables framework-level HTTP mapping. |
| **No framework lock-in for IDs** | `generateEntityId` uses NanoID under the hood, but the prefix scheme is Medusa's convention. |

---

## 3. Component Design

### 3.1 `MedusaService` factory

`MedusaService` is a function that accepts a map of MikroORM entity classes and returns an abstract class with auto-generated methods. The generated methods delegate to `MedusaInternalService`, which holds the actual MikroORM query logic.

```
MedusaService(models)
  └─ generates abstract class
       ├─ listProducts(filters, config, context)
       ├─ retrieveProduct(id, config, context)
       ├─ createProducts(data[], context)
       ├─ updateProducts(data[], context)
       ├─ deleteProducts(ids[], context)
       ├─ softDeleteProducts(ids[], context)
       └─ restoreProducts(ids[], context)
```

The method names are derived from the entity class name (e.g., `Product` → `listProducts`). This convention is enforced by TypeScript generics so that the generated method map is fully typed at compile time.

**Key design choice:** the factory pattern (not class inheritance) was chosen to support multiple entity models per service. A service managing both `Product` and `ProductVariant` gets methods for both from a single `extends MedusaService({ Product, ProductVariant })` declaration.

### 3.2 Decorator pipeline

Decorators apply in the following order when a method is called:

```
@InjectManager         → creates/reuses EntityManager from context
  @InjectTransactionManager  → wraps in a transaction if none exists
    @EmitEvents         → collects domain events, flushes after success
      @MedusaContext    → injects shared context into last parameter
        ← actual method body executes here
```

Each decorator modifies the `sharedContext` argument that is passed down the call chain. This allows a single `EntityManager` to span multiple service calls without leaking transaction state between requests.

### 3.3 `MedusaError`

```
MedusaError
├── Types.NOT_FOUND          → HTTP 404
├── Types.INVALID_DATA       → HTTP 400
├── Types.NOT_ALLOWED        → HTTP 400 (business rule violation)
├── Types.CONFLICT           → HTTP 409
├── Types.UNEXPECTED_STATE   → HTTP 500
├── Types.INVALID_ARGUMENT   → HTTP 400
└── Types.DB_ERROR           → HTTP 500
```

The framework layer catches `MedusaError` at the route handler level and maps `.type` to the appropriate HTTP status code. All other errors bubble up as 500.

### 3.4 `generateEntityId`

IDs are generated as `{prefix}_{nanoid(26)}`. The 26-character NanoID using an alphanumeric alphabet provides ~150 bits of entropy — collision probability is negligible even at Medusa Cloud scale.

The prefix is purely cosmetic and aids debugging (e.g., `prod_` vs `order_`), but is not parsed by any framework code.

---

## 4. Modules-SDK Sub-directory

The `modules-sdk/` sub-directory of utils contains the higher-level tools used specifically when authoring modules:

| File | Purpose |
|---|---|
| `definition.ts` | `Module()` helper — defines a module export |
| `define-link.ts` | `defineLink()` — declares cross-module relationships |
| `joiner-config-builder.ts` | Builds JoinerConfig for RemoteQuery integration |
| `medusa-internal-service.ts` | Low-level repository service wrapping MikroORM `EntityRepository` |
| `medusa-service.ts` | `MedusaService` factory (described above) |
| `module-provider.ts` | Helpers for provider modules (payment, notification, etc.) |
| `build-query.ts` | Translates `FindConfig<T>` to a MikroORM `FindOptions` object |

---

## 5. Dependency Graph

```
utils
├── requires (runtime): reflect-metadata, nanoid, MikroORM core
└── consumes (types only): @medusajs/framework/types
```

`utils` does not import from `modules-sdk` (the npm package) — it is the implementation that `modules-sdk` re-exports.

---

## 6. Testing Strategy

- **Unit tests**: placed in `dist/__tests__/` (compiled from `src/__tests__/`); Jest with `ts-jest`.
- **Decorator tests**: use simple in-memory entity managers to verify transaction wrapping without a real database.
- **`MedusaService` tests**: generate a sample service, exercise all generated methods against a SQLite in-memory database.
- **`MedusaError` tests**: assert correct `.type` and `.message` for each error kind.

---

## 7. Performance Considerations

- `MedusaService`-generated methods use a single `EntityManager` per request (injected via `@InjectManager`), avoiding N+1 connection pool hits.
- `buildQuery` uses MikroORM's `qb.select()` projection to avoid over-fetching columns not listed in `FindConfig.select`.
- `generateEntityId` calls NanoID's synchronous variant — no I/O, < 1 µs per call.

---

## 8. Open Questions / Future Work

- Evaluate replacing reflect-metadata with the TC39 Stage 3 decorators proposal once MikroORM supports it.
- Consider publishing `MedusaService` as a standalone package decoupled from MikroORM for custom ORM integrations.
- Add OpenTelemetry span instrumentation to the decorator pipeline.
