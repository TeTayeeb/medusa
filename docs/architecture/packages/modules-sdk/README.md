# @medusajs/modules-sdk

> Version: 2.15.4 · Package path: `packages/core/modules-sdk`

The **modules-sdk** package provides the bootstrapping and runtime infrastructure for every Medusa module. It is what module authors import to register, load, and link their modules. It is also what the Medusa application core uses to load all modules at startup.

---

## Installation

```bash
pnpm add @medusajs/modules-sdk
```

---

## What It Does

| Concern | Export |
|---|---|
| Module definition | `ModulesDefinition`, `MODULE_DEFINITIONS` |
| Module bootstrapping | `MedusaModule`, `MedusaApp` |
| Remote query | `RemoteQuery`, `remoteQueryObjectFromString` |
| Module linking | `defineLink` (re-exported from utils) |
| Container management | `createMedusaContainer` |
| Type utilities | `AbstractModuleService`, `ModuleBootstrapOptions` |

---

## Key Usage

### Define and register a custom module

```typescript
// src/modules/my-module/index.ts
import { Module } from "@medusajs/framework/utils"
import { MyModuleService } from "./service"

export default Module("my-module", {
  service: MyModuleService,
})
```

```typescript
// medusa-config.ts
import { defineConfig, Modules } from "@medusajs/framework/utils"

export default defineConfig({
  modules: [
    {
      resolve: "./src/modules/my-module",
      options: { apiKey: process.env.MY_API_KEY },
    },
  ],
})
```

### Bootstrap a module programmatically (e.g., in tests)

```typescript
import { MedusaModule, MedusaApp } from "@medusajs/modules-sdk"

const { modules } = await MedusaApp({
  modulesConfig: {
    [Modules.PRODUCT]: true,
    [Modules.INVENTORY]: true,
  },
  sharedResourcesConfig: {
    database: { clientUrl: process.env.DATABASE_URL },
  },
})

const productService = modules[Modules.PRODUCT]
const products = await productService.listProducts({})
```

### Remote Query — cross-module data fetching

```typescript
import { remoteQueryObjectFromString } from "@medusajs/modules-sdk"

// Build a query that spans Product and Inventory modules
const query = remoteQueryObjectFromString({
  entryPoint: "product",
  variables: { filters: { status: "published" } },
  fields: [
    "id",
    "title",
    "variants.id",
    "variants.sku",
    "variants.inventory_items.inventory.location_levels.available_quantity",
  ],
})

const [products, count] = await remoteQuery(query)
```

### Module linking — create a link between two modules

```typescript
import { defineLink } from "@medusajs/framework/utils"
import ProductModule from "@medusajs/product"
import CustomModule from "./modules/custom"

// Declares a 1-to-1 link between a product variant and a custom entity
export default defineLink(
  ProductModule.linkable.productVariant,
  CustomModule.linkable.customEntity
)
```

### `createMedusaContainer` — DI container for tests

```typescript
import { createMedusaContainer } from "@medusajs/modules-sdk"
import { asValue } from "awilix"

const container = createMedusaContainer()
container.register("logger", asValue(console))

// Use container in tests without a full Medusa server
```

---

## Module Loading Lifecycle

```
1. ModulesDefinition   — static registry of all built-in modules
2. MedusaModule.bootstrap()  — resolves module path, creates Awilix container
3. Loaders run         — db connection, model registration, container wiring
4. Service available   — container.resolve(moduleKey) returns service instance
5. JoinerConfig built  — module exposes linkable keys for RemoteQuery
```

---

## Directory Structure (dist)

```
dist/
├── definitions.js          MODULE_DEFINITIONS registry
├── link.js                 defineLink utilities
├── loaders/                Container and ORM loaders
├── medusa-app.js           Full Medusa app bootstrapper
├── medusa-module.js        Single-module bootstrapper
├── remote-query/           RemoteQuery implementation
├── types/                  ModuleBootstrapOptions, MigrationOptions
└── utils/                  Linking error helpers
```

---

## See Also

- [`@medusajs/framework/utils`](../utils/README.md) — `MedusaService`, `Module()`, decorators
- [`@medusajs/framework/types`](../types/README.md) — `ModuleDefinition`, `ModuleExports`
- [Module development guide](https://docs.medusajs.com/learn/fundamentals/modules)
