# arc42 Architecture Documentation — @medusajs/framework/utils

**Package:** `@medusajs/framework/utils`  
**Version:** 2.15.4  
**Template:** arc42 v8  

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/framework/utils` is the runtime foundation for all Medusa modules. It must:
- Reduce boilerplate by auto-generating CRUD operations from entity definitions.
- Enforce transactional correctness through declarative decorators.
- Provide a consistent error hierarchy understood by the HTTP layer.
- Supply utility functions (ID generation, validation, feature flags) that are safe to call anywhere.

### 1.2 Quality Goals

| Priority | Quality | Scenario |
|---|---|---|
| 1 | **Correctness** | Every write operation runs inside a database transaction, even if the caller forgets to provide one. |
| 2 | **Developer ergonomics** | A new module service with full CRUD can be written in < 10 lines. |
| 3 | **Consistency** | All module services behave identically for the same operation type. |
| 4 | **Testability** | Decorators can be tested with an in-memory entity manager. |

---

## 2. Architecture Constraints

- Must use MikroORM as the ORM for DB-backed operations.
- Decorators must use the legacy TypeScript experimental decorators API (Stage 2), because MikroORM relies on `reflect-metadata`.
- Must not import from `@medusajs/modules-sdk` (the npm package) to avoid circular dependencies.

---

## 3. System Scope and Context

```
  ┌──────────────┐      ┌──────────────────┐      ┌─────────────────┐
  │   modules    │────▶ │      utils       │────▶ │  MikroORM core  │
  │ (product,    │      │ MedusaService    │      │  EntityManager  │
  │  order, …)   │      │ Decorators       │      └─────────────────┘
  └──────────────┘      │ MedusaError      │
  ┌──────────────┐      │ generateEntityId │
  │  core-flows  │────▶ │ validateEmail    │
  │  (workflows) │      │ Modules enum     │
  └──────────────┘      └──────────────────┘
```

---

## 4. Solution Strategy

### 4.1 Factory over inheritance for services

`MedusaService` is a factory function, not a base class. This is intentional:

- A class can only extend one parent — a factory can compose from multiple entity models.
- The returned abstract class is specific to the provided models, giving fully typed method names.
- The factory is not instantiated at module load time; it only runs when the `extends` clause is evaluated.

### 4.2 Decorator composition for cross-cutting concerns

Instead of wrapping business logic in `try/finally` blocks, decorators handle:
- Opening / joining transactions (`@InjectTransactionManager`)
- Acquiring a read-only manager (`@InjectManager`)
- Collecting and flushing domain events (`@EmitEvents`)
- Injecting the shared context (`@MedusaContext`)

This keeps the method body focused on business logic.

### 4.3 Typed errors for HTTP mapping

`MedusaError.Types` is an enum whose values are matched by the framework's error middleware to produce correct HTTP status codes. Module code never needs to set status codes manually.

---

## 5. Building Block View

```
utils/
├── modules-sdk/
│   ├── medusa-service.ts       MedusaService factory
│   ├── medusa-internal-service.ts  Low-level repo service
│   ├── decorators/
│   │   ├── inject-manager.ts
│   │   ├── inject-transaction-manager.ts
│   │   ├── context-parameter.ts    (@MedusaContext)
│   │   ├── emit-events.ts
│   │   └── inject-shared-context.ts
│   ├── definition.ts           Module() helper
│   ├── define-link.ts
│   ├── joiner-config-builder.ts
│   └── build-query.ts
├── exceptions/
│   └── postgres-error.ts       isDuplicateError
├── event-bus/
│   └── (event name constants)
├── feature-flags/
│   └── is-feature-enabled.ts
├── dml/                        Data Modelling Layer
├── common/
│   └── (isObject, isString, promiseAll, …)
└── index.ts                    Root barrel export
```

---

## 6. Runtime View

### Scenario: Module service write method

```
1. HTTP route calls workflow step
2. Workflow step calls service.createProducts([data], sharedContext)
3. @InjectTransactionManager fires:
     - If sharedContext.transactionManager exists → reuse it
     - Else → begin new transaction on EntityManager
4. Method body executes with forked EntityManager
5. MikroORM Unit of Work flushes changes to DB
6. @EmitEvents fires:
     - Collects events saved to sharedContext.eventGroupId
     - Publishes to EventBus after commit
7. Transaction commits
8. Result returned to workflow step
```

---

## 7. Deployment View

`utils` is published as `@medusajs/utils` and re-exported from `@medusajs/framework/utils`. It has no environment-specific deployment concerns — it is a library, not a service.

---

## 8. Cross-Cutting Concepts

### 8.1 Entity ID convention

All IDs follow `{prefix}_{nanoid(26)}`. The prefix helps developers identify entity type from an ID alone during debugging. The NanoID portion provides collision-resistant uniqueness.

### 8.2 `buildQuery` translation

`FindConfig<T>` → MikroORM `FindOptions`:

| `FindConfig` field | MikroORM `FindOptions` field |
|---|---|
| `select` | `fields` |
| `relations` | `populate` |
| `take` | `limit` |
| `skip` | `offset` |
| `order` | `orderBy` |
| `withDeleted` | `filters: { softDelete: false }` |

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | `MedusaService` as factory function | Supports multiple models per service without multiple inheritance. |
| AD-02 | Legacy decorators | MikroORM v5/v6 requires `reflect-metadata`; Stage 3 decorators would break this. |
| AD-03 | `MedusaError.Types` enum | Maps errors to HTTP codes at one central point; module code stays clean. |
| AD-04 | NanoID for entity IDs | Shorter than UUID, URL-safe, no dependencies, cryptographically random. |

---

## 10. Risks and Technical Debt

- **Decorator API stability** — Stage 2 decorators may be deprecated. Migration to Stage 3 when MikroORM supports it.
- **MikroORM coupling** — `MedusaService` and `buildQuery` are tightly coupled to MikroORM semantics. A future abstraction layer would allow alternative ORMs.
