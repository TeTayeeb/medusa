# Software Design Document — @medusajs/modules-sdk

**Package:** `@medusajs/modules-sdk`  
**Version:** 2.15.4  
**Status:** Stable  
**Owner:** Medusa Core Team

---

## 1. Purpose and Scope

`modules-sdk` is the infrastructure layer that bootstraps, registers, and connects Medusa modules at runtime. It provides:

- The `MedusaModule` class for bootstrapping individual modules in isolation.
- The `MedusaApp` bootstrapper for loading a full set of modules with shared resources.
- The `RemoteQuery` engine for cross-module data fetching without direct service-to-service coupling.
- The `ModulesDefinition` registry that maps module keys to their default implementations.

---

## 2. Design Goals

| Goal | Decision |
|---|---|
| **Module isolation** | Each module runs in its own Awilix DI container. Cross-module calls go through RemoteQuery or workflow steps, never direct imports. |
| **Swappable implementations** | The `ModulesDefinition` registry provides defaults; any module can be overridden in `medusa-config.ts`. |
| **Lazy loading** | Modules are only bootstrapped when needed, enabling partial loads for migrations. |
| **Cross-module queries without coupling** | `RemoteQuery` uses JoinerConfig metadata to resolve relationships at query time. |

---

## 3. Component Design

### 3.1 Module Bootstrapping (`MedusaModule`)

When a module is bootstrapped:

```
MedusaModule.bootstrap(options)
  1. Resolve module path → load module exports (service class, loaders, migrations)
  2. Create Awilix container scoped to this module
  3. Run loaders:
     a. mikro-orm-connection-loader  → establish DB connection, run pending migrations
     b. container-loader-factory     → register service, repositories, providers
     c. custom loaders (if defined)  → e.g., seed data, external API warm-up
  4. Register module instance in MedusaModule._loadedModules map
  5. Return { [moduleKey]: serviceInstance }
```

The `migrationOnly` flag skips loader execution and only applies schema migrations, enabling zero-downtime deployments.

### 3.2 `MedusaApp` — full application bootstrap

`MedusaApp` orchestrates bootstrapping of all configured modules in parallel:

```typescript
const { modules, link, query } = await MedusaApp({
  modulesConfig: { ... },
  sharedResourcesConfig: { database: { clientUrl: "..." } },
})
```

It also:
- Creates the **Link module** — a join table module that stores cross-module entity relationships.
- Instantiates `RemoteQuery` with all loaded module `JoinerConfig`s.

### 3.3 `RemoteQuery` — cross-module graph fetching

RemoteQuery solves a core problem: modules are isolated, but product listings need inventory data, which lives in a different module. RemoteQuery is a lightweight federated query engine:

```
RemoteQuery.query(queryObject)
  1. Parse entryPoint and requested fields
  2. Resolve which modules own which fields using JoinerConfig
  3. Dispatch sub-queries to each module's service in parallel
  4. Join results using declared link keys
  5. Return merged result set
```

This is analogous to GraphQL's executor but simpler — no schema stitching, no resolvers, just declarative field-to-module mapping.

**JoinerConfig** is the metadata each module exposes:

```typescript
{
  serviceName: "productService",
  primaryKeys: ["id"],
  linkableKeys: { product_id: "Product", variant_id: "ProductVariant" },
  alias: [{ name: "product", args: { methodSuffix: "Products" } }],
}
```

### 3.4 `ModulesDefinition` registry

A static record mapping module keys to their default resolver paths:

```typescript
{
  [Modules.PRODUCT]:   { key: "product",   defaultPackage: "@medusajs/product",   ... },
  [Modules.ORDER]:     { key: "order",     defaultPackage: "@medusajs/order",     ... },
  [Modules.INVENTORY]: { key: "inventory", defaultPackage: "@medusajs/inventory", ... },
  // … 30+ entries
}
```

At runtime the framework reads this registry and uses the default path unless the user has overridden `resolve` in `medusa-config.ts`.

### 3.5 Link Module

The Link module is a synthetic module auto-generated from `defineLink` declarations. It creates join tables in the database and exposes them via RemoteQuery. It has no hand-written service — its schema and CRUD operations are generated entirely from the link definitions collected at startup.

---

## 4. Dependency Graph

```
modules-sdk
├── runtime deps: awilix, @medusajs/framework/utils, @medusajs/framework/types
└── peer deps:    @mikro-orm/postgresql (optional, only needed for DB-backed modules)
```

---

## 5. Error Handling

Bootstrap errors are non-recoverable and terminate the process. Module-level errors during request processing are wrapped in `MedusaError` by the module service and propagate to the HTTP layer.

---

## 6. Testing

- `createMedusaContainer()` creates a minimal Awilix container for unit tests.
- Integration tests in `packages/core/modules-sdk/dist/__tests__/` bootstrap real modules against a PostgreSQL test database.
- Isolation is enforced: tests that bootstrap `MedusaApp` must call `MedusaModule.clearInstances()` in `afterAll` to avoid state leakage between test suites.

---

## 7. Open Questions / Future Work

- Consider moving `RemoteQuery` to a dedicated package to reduce `modules-sdk` bundle size.
- Investigate lazy-loading JoinerConfigs to speed up cold start on serverless deployments.
- Add a dev-mode warning when a query crosses more than N module boundaries (performance hint).
