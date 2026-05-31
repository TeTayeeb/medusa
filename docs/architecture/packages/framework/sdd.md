# Software Design Document — `@medusajs/framework`

**Version:** 2.15.4  
**Status:** Released  
**Authors:** Medusa Core Team

---

## 1. Purpose & Scope

This document describes the internal design of the `@medusajs/framework` package — the Medusa application runtime. It covers architectural decisions, component responsibilities, data flows, and extension points for contributors and advanced integrators.

---

## 2. Goals & Non-Goals

### Goals
- Provide a zero-configuration bootstrap experience for new Medusa applications.
- Enable file-convention-based extension (routes, jobs, subscribers, middlewares).
- Expose stable, typed interfaces for every sub-system.
- Support hot-module reload for development.

### Non-Goals
- Implement commerce domain logic (that belongs in module packages).
- Replace or abstract Express — it remains the transport layer.
- Provide a UI or admin dashboard.

---

## 3. Component Architecture

```
MedusaAppLoader
├── ConfigLoader         — parse medusa-config.ts
├── ContainerBuilder     — initialise Awilix container
├── DatabaseLoader       — pg-connection-loader (Knex pool)
├── ModulesLoader        — load all module services into container
├── LinkModulesLoader    — load cross-module link definitions
├── ApiLoader (HTTP)
│   ├── routes-finder    — walk FS for route.ts files
│   ├── routes-sorter    — priority ordering
│   ├── routes-loader    — dynamic import → register handlers
│   └── middleware-file-loader — register middlewares.ts
├── JobsLoader           — scan src/jobs/** → schedule cron tasks
├── SubscribersLoader    — scan src/subscribers/** → bind event handlers
├── PoliciesLoader       — load auth / CORS / rate-limit middleware
└── TelemetryReporter    — optional usage telemetry
```

---

## 4. Sub-system Design

### 4.1 HTTP Layer

**Design choice:** filesystem-as-router.

Route files are discovered by `routes-finder` using a recursive directory walk. The sorter applies a deterministic priority algorithm:

1. Exact static segments (`/admin/products`)
2. Named parameters (`/admin/products/:id`)
3. Wildcard catch-all (`/admin/*`)

Each `route.ts` must export at least one HTTP verb as a named export:

```typescript
export const GET: RouteHandler = async (req, res) => { … }
export const POST: RouteHandler = async (req, res) => { … }
```

The `ApiLoader` wraps each handler through optional `traceRoute` / `traceMiddleware` interceptors for OpenTelemetry instrumentation without coupling route code to observability libraries.

**Request typing:** `MedusaRequest` extends Express `Request` with:
- `scope: MedusaContainer` — request-scoped DI container
- `filterableFields: Record<string, unknown>` — parsed filter params
- `queryConfig: { fields, pagination }` — validated query config
- `auth_context` — decoded JWT/API-key identity

### 4.2 Dependency Injection Container

Awilix is configured with three scopes:

| Scope | Lifetime | Examples |
|---|---|---|
| `SINGLETON` | process lifetime | module services, logger, DB pool |
| `SCOPED` | request lifetime | transaction manager, request metadata |
| `TRANSIENT` | per-resolution | one-off utility instances |

Module services are registered under their canonical key (e.g., `productModuleService`). The container key is derived from the module's `Modules` enum value.

`ContainerRegistrationKeys` enumerates special well-known keys:
- `QUERY` → cross-module query engine
- `LOGGER` → structured logger
- `CONFIG_MODULE` → parsed config
- `REMOTE_LINK` → module linker

### 4.3 Database Layer

A single Knex connection pool (configured via `projectConfig.databaseUrl`) is shared among all modules. Each MikroORM entity manager creates a fork per request, ensuring transaction isolation without pool exhaustion.

Migration lifecycle:

```
MedusaAppLoader.runModulesMigrations({ action: "run" })
  └─ for each module:
       MikroORM.getMigrator().up()  // applies pending migrations
```

The `mikro-orm-cli` wrapper re-exports MikroORM's CLI entrypoint configured with Medusa's `tsconfig` path mappings so that `npx medusa db:migrate` works without manual setup.

### 4.4 Module Loader

`MedusaAppLoader` calls `mergeDefaultModules()` to combine the 30+ built-in modules with any user-provided overrides. Each module declaration is:

```typescript
type InternalModuleDeclaration = {
  scope: "internal"
  resources: "shared"   // uses the shared pg pool
  resolve: string        // npm package or local path
  options?: Record<string, unknown>
}
```

Modules register their services into the shared container. Cross-module data access is mediated by the **Module Link** sub-system, which generates join tables and injects a `RemoteLink` resolver.

### 4.5 Policies

Request policies are declared as `{ resource, operation }` tuples on `MiddlewareRoute` descriptors. At load time, `PoliciesLoader` resolves each tuple to a concrete middleware:

- **`authenticate`** — verifies JWT bearer token or API key, populates `req.auth_context`
- **`corsPolicy`** — checks `Origin` against configured allowlists
- **`rateLimitPolicy`** — in-memory token-bucket per IP (configurable)

### 4.6 Zod Validation Integration

`validate-body(schema)` and `validate-query(schema)` are middleware factories that:

1. Parse the incoming payload with the provided Zod schema.
2. On failure, respond `422 Unprocessable Entity` with a structured error object listing each field violation.
3. On success, replace `req.body` / `req.query` with the validated, typed value.

---

## 5. Error Handling Strategy

All errors thrown inside framework internals extend `MedusaError`:

```typescript
throw new MedusaError(
  MedusaError.Types.NOT_FOUND,
  `Module "xyz" could not be resolved`
)
```

The Express error handler middleware catches `MedusaError` instances and maps them to HTTP status codes (`404`, `400`, `403`, etc.). Unrecognised errors map to `500`.

---

## 6. Extension Points

| Extension Point | Convention | Description |
|---|---|---|
| Custom API routes | `src/api/**/route.ts` | Named HTTP verb exports |
| Custom middlewares | `src/api/**/middlewares.ts` | `defineMiddlewares()` default export |
| Background jobs | `src/jobs/*.ts` | Default fn + `config.schedule` |
| Event subscribers | `src/subscribers/*.ts` | Default fn + `config.event` |
| Custom modules | `medusa-config.ts#modules` | DI-registered service |
| Workflow hooks | Via `@medusajs/core-flows` hooks API | Pre/post step injection |

---

## 7. Performance Considerations

- Route files are discovered and cached once at startup; no filesystem scan per request.
- The Awilix container uses a proxy-based resolution cache, avoiding repeat lookups.
- The Knex connection pool defaults to `min: 2, max: 10` and is shared across all modules to prevent N×pool connection exhaustion.
- Telemetry is fire-and-forget; it never blocks the request path.

---

## 8. Security Considerations

- Authentication is enforced at the route level via `policies` before business logic executes.
- Zod schemas strip unknown fields from request bodies by default (`strip` mode).
- The DI container is request-scoped; cross-request data leakage is structurally prevented.
- `RestrictedFields` utility ensures sensitive fields (e.g., password hashes) are never included in API responses without explicit opt-in.
