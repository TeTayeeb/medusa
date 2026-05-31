# SpecKit — @medusajs/modules-sdk

**Package:** `@medusajs/modules-sdk`  
**Version:** 2.15.4  
**Document type:** Specification & Test Contracts  

---

## 1. Overview

This SpecKit defines the behavioural contracts for the `modules-sdk` package. It covers module bootstrapping, RemoteQuery, module links, and the `ModulesDefinition` registry.

---

## 2. Functional Specifications

### SPEC-MSDK-01 — `MedusaModule.bootstrap` loads a module

**Description:** Given a valid `moduleKey` and `modulePath`, `MedusaModule.bootstrap` must:
1. Resolve and `require` the module exports.
2. Establish a database connection.
3. Run pending migrations.
4. Register the service in the module's DI container.
5. Return an object `{ [moduleKey]: serviceInstance }`.

**Contract:**
```typescript
const { productService } = await MedusaModule.bootstrap({
  moduleKey: Modules.PRODUCT,
  defaultPath: "@medusajs/product",
  declaration: { resolve: "@medusajs/product" },
  sharedContainer: appContainer,
})

expect(productService).toBeDefined()
expect(typeof productService.listProducts).toBe("function")
```

**Failure mode:** If the module path cannot be resolved, throws with a message identifying the missing module.

---

### SPEC-MSDK-02 — Module isolation

**Description:** Two modules bootstrapped with `MedusaApp` must not share a DI container. Registering a value in one container must not affect the other.

**Contract:**
```typescript
const { modules } = await MedusaApp({ modulesConfig: { product: true, order: true } })
const productContainer = modules[Modules.PRODUCT].__container__
const orderContainer   = modules[Modules.ORDER].__container__

expect(productContainer).not.toBe(orderContainer)
```

---

### SPEC-MSDK-03 — `RemoteQuery` cross-module field resolution

**Description:** A `RemoteQuery` call with `entryPoint: "product"` and `fields: ["variants.inventory_items.inventory.location_levels.available_quantity"]` must return data populated from both the Product module and the Inventory module without requiring a direct service-to-service import.

**Contract:**
```typescript
const [products] = await remoteQuery({
  entryPoint: "product",
  variables: { filters: { id: ["prod_123"] } },
  fields: ["id", "title", "variants.id", "variants.inventory_items.inventory.id"],
})

expect(products[0].variants[0].inventory_items[0].inventory.id).toBeDefined()
```

**Failure mode:** If a field cannot be resolved to any module's JoinerConfig, `RemoteQuery` throws with a message listing the unresolvable field path.

---

### SPEC-MSDK-04 — `defineLink` creates a queryable relationship

**Description:** After `defineLink(ProductModule.linkable.productVariant, CustomModule.linkable.customEntity)` and running migrations, the Link Module must create a join table and expose the relationship via RemoteQuery.

**Contract:**
```typescript
// After linking variant_id → custom_entity_id:
const [variants] = await remoteQuery({
  entryPoint: "productVariant",
  fields: ["id", "custom_entity.id", "custom_entity.name"],
})
expect(variants[0].custom_entity).toBeDefined()
```

---

### SPEC-MSDK-05 — `MedusaModule.clearInstances` resets state

**Description:** After `MedusaModule.clearInstances()`, `MedusaModule.getLoadedModules()` must return an empty array.

```typescript
await MedusaApp({ modulesConfig: { product: true } })
expect(MedusaModule.getLoadedModules().length).toBeGreaterThan(0)

MedusaModule.clearInstances()
expect(MedusaModule.getLoadedModules().length).toBe(0)
```

**Use case:** Required in `afterAll` blocks of integration test suites to prevent state leakage.

---

### SPEC-MSDK-06 — `migrationOnly` bootstrap skips loaders

**Description:** When `migrationOnly: true` is passed to `MedusaModule.bootstrap`, the container loader must not run and the service must not be registered.

**Contract:** The returned module object must not have a callable service; attempting to resolve the service from the container throws `AWLIX_ERR_NAME_NOT_REGISTERED`.

---

### SPEC-MSDK-07 — `ModulesDefinition` provides defaults for all built-in modules

**Description:** For every value in the `Modules` enum, `ModulesDefinition[moduleKey]` must exist and have a `defaultPackage` field pointing to a published npm package.

```typescript
import { Modules } from "@medusajs/framework/utils"
import { ModulesDefinition } from "@medusajs/modules-sdk"

for (const key of Object.values(Modules)) {
  expect(ModulesDefinition[key]).toBeDefined()
  expect(ModulesDefinition[key].defaultPackage).toBeTruthy()
}
```

---

## 3. Non-Functional Specifications

### SPEC-MSDK-NF-01 — Parallel module loading

**Contract:** `MedusaApp` must bootstrap independent modules concurrently. The total bootstrap time must not exceed `max(individual_module_time) + 500ms` overhead.

### SPEC-MSDK-NF-02 — RemoteQuery join performance

**Contract:** A RemoteQuery call that spans two modules must complete in under 200 ms (P95) in a test environment with local PostgreSQL and pre-seeded data of 1,000 records per module.

---

## 4. Test Matrix

| Spec | Test type | Location |
|---|---|---|
| SPEC-MSDK-01 | Integration | `dist/__tests__/module-bootstrap.spec.ts` |
| SPEC-MSDK-02 | Integration | `dist/__tests__/module-isolation.spec.ts` |
| SPEC-MSDK-03 | Integration | `integration-tests/http/__tests__/remote-query.spec.ts` |
| SPEC-MSDK-04 | Integration | `integration-tests/http/__tests__/module-links.spec.ts` |
| SPEC-MSDK-05 | Unit | `dist/__tests__/clear-instances.spec.ts` |
| SPEC-MSDK-06 | Unit | `dist/__tests__/migration-only.spec.ts` |
| SPEC-MSDK-07 | Unit | `dist/__tests__/modules-definition.spec.ts` |

---

## 5. Glossary

| Term | Definition |
|---|---|
| **JoinerConfig** | Metadata object each module exposes so RemoteQuery can resolve field paths |
| **Link Module** | Auto-generated module that stores cross-module entity relationships |
| **MedusaApp** | Bootstrapper that loads all configured modules with shared resources |
| **RemoteQuery** | Cross-module graph query engine; analogous to a federated GraphQL executor |
| **Awilix** | The DI container library used for dependency injection within each module |
