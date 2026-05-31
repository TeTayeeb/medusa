# `@medusajs/framework` — Core Runtime Package

**Version:** 2.15.4  
**License:** MIT  
**Category:** Core Infrastructure

---

## Overview

`@medusajs/framework` is the central runtime package of Medusa v2. It bootstraps the entire application — wiring together the HTTP server, dependency-injection container, database connection, module loader, background jobs, event subscribers, workflow context, and request validation into a unified, production-ready foundation.

Every Medusa application depends on this package at runtime. It exposes well-typed sub-path exports so that consumers pull only what they need (e.g., `@medusajs/framework/http`, `@medusajs/framework/workflows-sdk`) without importing the full bundle.

---

## Sub-system Breakdown

### HTTP Layer

Built on Express.js, the HTTP layer performs filesystem-based route discovery at startup.

- **`ApiLoader`** scans one or more source directories for `route.ts` files and `middlewares.ts` files.
- **`routes-finder`** walks the filesystem to collect all matching route files.
- **`routes-loader`** dynamically imports each file and registers `GET`, `POST`, `PUT`, `PATCH`, `DELETE` named exports as Express handlers.
- **`routes-sorter`** resolves route priority (static paths before parameterised paths before wildcard paths) to prevent shadowing.
- **`middleware-file-loader`** picks up `middlewares.ts` files alongside routes and injects them at the correct Express layer.

### DI Container

An [Awilix](https://github.com/jeffijoe/awilix) IoC container governs every service lifetime.

- **Singleton scope** — module services, database connections, logger.
- **Request scope** — per-request database transaction managers, request-scoped overrides.
- The `MedusaContainer` type extends Awilix's `AwilixContainer` with Medusa-specific resolution helpers.

### Database

Medusa uses [MikroORM](https://mikro-orm.io/) with PostgreSQL.

- **`pg-connection-loader`** creates a shared `Knex` connection pool that is injected into every module.
- **`MedusaAppLoader.runModulesMigrations()`** drives `run`, `revert`, and `generate` migration actions against all registered modules in one transaction.
- The `mikro-orm-cli` wrapper bridges Medusa's config format with MikroORM's CLI expectations.

### Application Loader (`MedusaAppLoader`)

`MedusaAppLoader` is the root orchestrator. It:

1. Merges default module declarations with user config.
2. Prepares shared resources (DB pool, logger).
3. Resolves and loads all modules via `runModulesLoader()`.
4. Supports hot-module reload via `reloadSingleModule()`.
5. Exposes a `getLinksExecutionPlanner()` for cross-module link migrations.

### Config

A `medusa-config.ts` (or `.js`) file at the project root is loaded once at startup. Environment variables are parsed and merged into a typed `ConfigModule` shape covering `projectConfig`, `modules`, `plugins`, `featureFlags`, and `admin` settings.

### Jobs & Subscribers

Files placed under `src/jobs/*.ts` and `src/subscribers/*.ts` are auto-discovered by file-convention loaders. Jobs receive a cron-expression export; subscribers export an event name and handler.

### Policies

The `policies` sub-system provides route-level middleware that checks **authentication** (JWT bearer, API key), **CORS** origins, and **rate-limiting** caps before requests reach route handlers.

### Zod Validation

`validate-body` and `validate-query` utilities wrap Zod schemas into Express middleware, producing typed `req.body` and `req.query` with standardised 422 error shapes.

---

## Installation

```bash
# Already included in every Medusa project
yarn add @medusajs/framework
```

---

## Quick-Start: Custom Route

```typescript
// src/api/store/hello/route.ts
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"

export const GET = async (
  req: MedusaRequest,
  res: MedusaResponse
) => {
  res.json({ message: "Hello from Medusa!" })
}
```

## Quick-Start: Custom Middleware

```typescript
// src/api/middlewares.ts
import { defineMiddlewares } from "@medusajs/framework/http"

export default defineMiddlewares({
  routes: [
    {
      matcher: "/store/*",
      middlewares: [
        (req, res, next) => {
          req.headers["x-store"] = "true"
          next()
        },
      ],
    },
  ],
})
```

## Quick-Start: Background Job

```typescript
// src/jobs/daily-sync.ts
import { MedusaContainer } from "@medusajs/framework"

export default async function dailySync(container: MedusaContainer) {
  const logger = container.resolve("logger")
  logger.info("Running daily sync job…")
}

export const config = {
  name: "daily-sync",
  schedule: "0 2 * * *", // every day at 02:00
}
```

---

## Key Exports

| Sub-path | Notable exports |
|---|---|
| `@medusajs/framework/http` | `MedusaRequest`, `MedusaResponse`, `AuthenticatedMedusaRequest`, `ApiLoader`, `defineMiddlewares` |
| `@medusajs/framework/utils` | `MedusaError`, `MedusaService`, `InjectManager`, `MedusaContext`, `Modules` |
| `@medusajs/framework/workflows-sdk` | `createStep`, `createWorkflow`, `StepResponse`, `transform`, `when`, `parallelize` |
| `@medusajs/framework` (root) | `MedusaAppLoader`, `container`, `MEDUSA_CLI_PATH` |

---

## Related Packages

- [`@medusajs/core-flows`](../core-flows/README.md) — pre-built workflows built on this framework
- [`@medusajs/workflows-sdk`](../workflows-sdk/README.md) — step/workflow composition primitives
- [`@medusajs/types`](https://github.com/medusajs/medusa) — shared TypeScript interfaces
