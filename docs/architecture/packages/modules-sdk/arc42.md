# arc42 Architecture Documentation — @medusajs/modules-sdk

**Package:** `@medusajs/modules-sdk`  
**Version:** 2.15.4  
**Template:** arc42 v8  

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/modules-sdk` is the infrastructure that makes the Medusa module system work at runtime. It must:
- Bootstrap any module from a path string and a configuration object.
- Coordinate the loading of a full set of modules with shared database resources.
- Enable cross-module data fetching without introducing direct service-to-service coupling.
- Allow modules to declare relationships to each other (links) that are persisted and queryable.

### 1.2 Quality Goals

| Priority | Quality | Scenario |
|---|---|---|
| 1 | **Isolation** | A crash in the Payment module must not affect the Product module. |
| 2 | **Replaceability** | Any built-in module can be swapped for a custom implementation in config. |
| 3 | **Query efficiency** | RemoteQuery fetches cross-module data in parallel, not sequentially. |
| 4 | **Testability** | Any module can be bootstrapped in isolation for integration tests. |

---

## 2. Architecture Constraints

- Modules communicate exclusively through `RemoteQuery` or workflow steps — never via direct `import` of another module's service.
- The DI container is Awilix. Other IoC containers are not supported.
- PostgreSQL is the only supported database for production use.

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────────┐
│                     Medusa Application                     │
│                                                            │
│  ┌─────────────┐    ┌──────────────────────────────────┐  │
│  │ medusa-config│──▶│          modules-sdk             │  │
│  └─────────────┘    │  ┌──────────┐  ┌─────────────┐  │  │
│                     │  │MedusaApp │  │RemoteQuery  │  │  │
│  ┌─────────────┐    │  └──────────┘  └─────────────┘  │  │
│  │  API Routes │    │  ┌──────────────────────────┐   │  │
│  │  Workflows  │──▶ │  │     Module Containers    │   │  │
│  └─────────────┘    │  │  [product] [order] [...] │   │  │
│                     │  └──────────────────────────┘   │  │
│                     └──────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

### 4.1 Per-module Awilix containers

Each module gets its own Awilix DI container scoped from the shared application container. This provides:
- **Namespace isolation** — two modules can register a service under the same key without conflict.
- **Lazy unregistration** — a module can be unloaded without affecting the rest of the application.

### 4.2 Declarative module definition

Modules export a plain object (`ModuleExports`) that describes the service class, optional loaders, optional migrations, and the JoinerConfig. `modules-sdk` reads this object at bootstrap time and wires everything together. Module authors never write bootstrap code manually.

### 4.3 JoinerConfig-driven RemoteQuery

RemoteQuery does not require a GraphQL schema or special resolvers. Each module declares its `JoinerConfig` — a plain object listing its primary keys, alias names, and linkable keys. `RemoteQuery` uses these configs to build a resolution plan for any given query object.

### 4.4 Link Module as a first-class entity

Cross-module links are not stored in either linked module's database. They live in a separate auto-generated "Link Module" with its own join tables. This keeps module databases independent and makes link management transactional.

---

## 5. Building Block View

```
modules-sdk/
├── medusa-app.ts       MedusaApp() — full application bootstrapper
├── medusa-module.ts    MedusaModule — single-module bootstrapper
│                         bootstrap(), clearInstances(), getLoadedModules()
├── definitions.ts      MODULE_DEFINITIONS registry
├── loaders/
│   ├── container-loader-factory.ts   Register service + repos in container
│   ├── mikro-orm-connection-loader.ts  DB connection + migrations
│   └── load-models.ts                 Discover MikroORM entities
├── remote-query/
│   ├── remote-query.ts    RemoteQuery engine
│   ├── remote-joiner.ts   Cross-module join resolver
│   └── to-remote-query.ts remoteQueryObjectFromString helper
├── link.ts             defineLink, LinkModuleDefinition
├── types/
│   └── index.ts        ModuleBootstrapOptions, MigrationOptions
└── utils/
    └── linking-error.ts
```

---

## 6. Runtime View

### Cold start sequence

```
process start
  │
  ├─ loadConfig(medusa-config.ts)
  ├─ merge user config with ModulesDefinition defaults
  │
  ├─ MedusaApp({ modulesConfig, sharedResourcesConfig })
  │   ├─ for each module (parallel):
  │   │   ├─ resolve module path
  │   │   ├─ create Awilix container
  │   │   ├─ run mikro-orm-connection-loader (connect, migrate)
  │   │   └─ run container-loader-factory (register service)
  │   │
  │   ├─ collect all JoinerConfigs
  │   ├─ bootstrap Link Module (generate join tables)
  │   └─ create RemoteQuery with all JoinerConfigs
  │
  └─ application ready
```

### RemoteQuery execution

```
remoteQuery({ entryPoint: "product", fields: ["id", "variants.inventory_items.*"] })
  │
  ├─ parse fields → identify owning modules: product, inventory
  ├─ dispatch sub-query to product module service (listProducts)
  ├─ dispatch sub-query to link module (resolve variant ↔ inventory_item links)
  ├─ dispatch sub-query to inventory module service (listInventoryItems)
  ├─ join results by primary/foreign keys
  └─ return merged data
```

---

## 7. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | Awilix for DI | Existing Medusa v1 convention; mature, scope-aware, TS-compatible. |
| AD-02 | RemoteQuery instead of shared service imports | Enforces module isolation; enables query-level optimisation. |
| AD-03 | Auto-generated Link Module | No hand-written join-table code; link schema evolves with `defineLink` declarations. |
| AD-04 | `ModulesDefinition` registry | Enables zero-config use of built-in modules; easy override in config. |

---

## 8. Risks and Technical Debt

- **RemoteQuery N+1 risk** — if a field list triggers many small sub-queries, performance degrades. Mitigation: `DataLoader`-style batching is on the roadmap.
- **Awilix as a constraint** — some edge cases (e.g., async factory registrations) are awkward in Awilix. Evaluated alternatives are tracked in the ADR log.
