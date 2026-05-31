# SpecKit — @medusajs/framework/utils

**Package:** `@medusajs/framework/utils`  
**Version:** 2.15.4  
**Document type:** Specification & Test Contracts  

---

## 1. Overview

This SpecKit defines the behavioural contracts for the `utils` package. It covers `MedusaService`, the decorator suite, `MedusaError`, and utility functions. Each specification is independently verifiable.

---

## 2. Functional Specifications

### SPEC-UTILS-01 — `MedusaService` method generation

**Description:** Calling `MedusaService({ Product })` must generate a class with all standard CRUD methods for `Product`.

**Contract:**
```typescript
class ProductService extends MedusaService({ Product }) {}
const svc = new ProductService(/* ... */)

// All of these must exist and be callable:
await svc.listProducts({}, {})
await svc.listAndCountProducts({}, {})
await svc.retrieveProduct("id", {})
await svc.createProducts([{ title: "T" }])
await svc.updateProducts([{ id: "x", title: "U" }])
await svc.deleteProducts(["id"])
await svc.softDeleteProducts(["id"])
await svc.restoreProducts(["id"])
```

**Failure mode:** If `Product` is not a valid MikroORM entity class, the factory throws at class definition time with a descriptive error.

---

### SPEC-UTILS-02 — `@InjectTransactionManager` wraps writes in a transaction

**Description:** Any method decorated with `@InjectTransactionManager` must execute its body within a MikroORM transaction. If a transaction is already active in `sharedContext.transactionManager`, the decorator must join it rather than create a nested one (unless `enableNestedTransactions: true`).

**Test scenario:**
```typescript
// Arrange: service with decorated method
class MyService extends MedusaService({ Item }) {
  @InjectTransactionManager()
  async doWrite(@MedusaContext() ctx: Context = {}) {
    expect(ctx.transactionManager).toBeDefined()
    // Perform write
  }
}

// Act: call without providing a context
await service.doWrite()
// Assert: method executed inside a transaction (no error thrown)
```

**Failure mode:** If the database connection fails, the decorator propagates the error after rolling back.

---

### SPEC-UTILS-03 — `MedusaError` type-to-HTTP mapping

**Description:** Each `MedusaError.Types` value must map to a specific HTTP status code at the framework error middleware.

| `MedusaError.Types` | Expected HTTP status |
|---|---|
| `NOT_FOUND` | 404 |
| `INVALID_DATA` | 400 |
| `NOT_ALLOWED` | 400 |
| `CONFLICT` | 409 |
| `UNEXPECTED_STATE` | 500 |
| `INVALID_ARGUMENT` | 400 |
| `DB_ERROR` | 500 |

**Test:** Each HTTP integration test that triggers a known error checks the response status code against this table.

---

### SPEC-UTILS-04 — `generateEntityId` format

**Description:** `generateEntityId(undefined, "prod")` must return a string matching `/^prod_[A-Za-z0-9]{26}$/`.

```typescript
const id = generateEntityId(undefined, "prod")
expect(id).toMatch(/^prod_[A-Za-z0-9]{26}$/)

// Calling with an existing ID must return it unchanged
const existing = "prod_01HXFOO"
expect(generateEntityId(existing, "prod")).toBe(existing)
```

---

### SPEC-UTILS-05 — `validateEmail` behaviour

**Description:**

| Input | Expected behaviour |
|---|---|
| `"user@example.com"` | Returns without error |
| `"not-an-email"` | Throws `MedusaError` with type `INVALID_DATA` |
| `""` (empty string) | Throws `MedusaError` with type `INVALID_DATA` |
| `null` / `undefined` | Does not throw (guard returns early) |

---

### SPEC-UTILS-06 — `@EmitEvents` deferred emission

**Description:** Events collected during a method decorated with `@EmitEvents` must only be published **after** the transaction commits successfully. If the transaction rolls back, no events must be emitted.

**Test scenario:**
1. Call a method that saves an event and then throws an error.
2. Assert that `EventBus.emit` was never called.

---

### SPEC-UTILS-07 — `buildQuery` translation fidelity

**Description:** `buildQuery(filters, config)` must produce a MikroORM `FindOptions` object where:

| `FindConfig` input | MikroORM output |
|---|---|
| `select: ["id", "title"]` | `fields: ["id", "title"]` |
| `relations: ["variants"]` | `populate: ["variants"]` |
| `take: 20` | `limit: 20` |
| `skip: 40` | `offset: 40` |
| `order: { title: "ASC" }` | `orderBy: { title: "ASC" }` |
| `withDeleted: true` | soft-delete filter disabled |

---

## 3. Non-Functional Specifications

### SPEC-UTILS-NF-01 — `generateEntityId` performance

**Contract:** `generateEntityId` must complete in under 100 µs per call (synchronous, no I/O).

### SPEC-UTILS-NF-02 — Decorator overhead

**Contract:** The combined overhead of `@InjectManager` + `@MedusaContext` on a no-op method must be less than 1 ms.

---

## 4. Test Matrix

| Spec | Test type | Location |
|---|---|---|
| SPEC-UTILS-01 | Unit (Jest) | `dist/__tests__/modules-sdk/medusa-service.spec.ts` |
| SPEC-UTILS-02 | Unit (Jest + in-memory EM) | `dist/__tests__/modules-sdk/decorators/inject-transaction-manager.spec.ts` |
| SPEC-UTILS-03 | HTTP integration | `integration-tests/http/__tests__/error-handling.spec.ts` |
| SPEC-UTILS-04 | Unit (Jest) | `dist/__tests__/common/generate-entity-id.spec.ts` |
| SPEC-UTILS-05 | Unit (Jest) | `dist/__tests__/common/validate-email.spec.ts` |
| SPEC-UTILS-06 | Unit (Jest) | `dist/__tests__/modules-sdk/decorators/emit-events.spec.ts` |
| SPEC-UTILS-07 | Unit (Jest) | `dist/__tests__/modules-sdk/build-query.spec.ts` |

---

## 5. Glossary

| Term | Definition |
|---|---|
| **EntityManager** | MikroORM's unit-of-work manager; tracks entity changes and flushes to DB |
| **JoinerConfig** | Metadata object describing a module's linkable keys for RemoteQuery |
| **Soft delete** | Marks a record with `deleted_at` instead of removing it from the DB |
| **NanoID** | A compact, URL-safe, cryptographically random ID generator |
