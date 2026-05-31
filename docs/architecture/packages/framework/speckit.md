# SpecKit — `@medusajs/framework`

**Version:** 2.15.4  
**Spec Type:** Behavioural + Interface Specification

---

## 1. Package Identity

| Field | Value |
|---|---|
| Package name | `@medusajs/framework` |
| NPM scope | `@medusajs` |
| Source path | `packages/core/framework` |
| Main entry | `dist/index.js` |
| Types entry | `dist/index.d.ts` |
| Sub-paths | `http`, `utils`, `workflows-sdk`, `database`, `config`, `mikro-orm/*` |

---

## 2. Public API Contract

### 2.1 `MedusaAppLoader`

```typescript
class MedusaAppLoader {
  constructor(options?: {
    container?: MedusaContainer
    customLinksModules?: RegisterModuleJoinerConfig | RegisterModuleJoinerConfig[]
    medusaConfigPath?: string
    cwd?: string
  })

  runModulesMigrations(options?:
    | { action: "run"; allOrNothing?: boolean }
    | { action: "revert" | "generate"; moduleNames: string[] }
  ): Promise<void>

  getLinksExecutionPlanner(): Promise<ILinkMigrationsPlanner>
  runModulesLoader(): Promise<void>
  reloadSingleModule(options: {
    moduleKey: string
    serviceName: string
  }): Promise<LoadedModule | null>
  load(): Promise<MedusaAppOutput>
}
```

**Behaviour contract:**
- `load()` MUST complete all loaders in the correct dependency order before resolving.
- `runModulesMigrations({ action: "run" })` MUST be idempotent — re-running on an already-migrated database MUST be a no-op.
- `reloadSingleModule()` MUST NOT restart other modules or drop existing HTTP connections.

### 2.2 HTTP Request/Response Types

```typescript
interface MedusaRequest<Body = unknown> extends Request {
  scope: MedusaContainer
  auth_context?: AuthContext
  filterableFields: Record<string, unknown>
  queryConfig: {
    fields: string[]
    pagination: { skip: number; take: number; order?: Record<string, string> }
  }
  body: Body
}

interface MedusaResponse<Body = unknown> extends Response {
  json(body: Body): this
}

type AuthenticatedMedusaRequest<Body = unknown> =
  MedusaRequest<Body> & { auth_context: AuthContext }
```

### 2.3 `defineMiddlewares`

```typescript
function defineMiddlewares(config: MiddlewaresConfig): MiddlewaresConfig

type MiddlewaresConfig = {
  errorHandler?: false | MedusaErrorHandlerFunction
  routes?: MiddlewareRoute[]
}

type MiddlewareRoute = {
  matcher: string | RegExp       // path pattern
  methods?: MiddlewareVerb[]     // defaults to all verbs
  middlewares?: MiddlewareFunction[]
  bodyParser?: false | { sizeLimit?: string | number; preserveRawBody?: boolean }
  additionalDataValidator?: ZodRawShape
  policies?: PolicyDescriptor | PolicyDescriptor[]
}
```

---

## 3. Filesystem Conventions

| Convention | Path | Required exports |
|---|---|---|
| API Route | `src/api/**/route.ts` | At least one of: `GET POST PUT PATCH DELETE` |
| Middlewares | `src/api/**/middlewares.ts` | `default: MiddlewaresConfig` |
| Background Job | `src/jobs/*.ts` | `default: (container) => Promise<void>` + `config: { name, schedule }` |
| Event Subscriber | `src/subscribers/*.ts` | `default: (args, container) => Promise<void>` + `config: { event }` |

---

## 4. Invariants

1. **Route uniqueness:** Two `route.ts` files at the same filesystem path MUST NOT export the same HTTP verb. If they do, the last-loaded file wins (undefined behaviour — treat as a configuration error).
2. **Container resolution:** Calling `req.scope.resolve(key)` for an unregistered key MUST throw `AwilixResolutionError` synchronously.
3. **Migration atomicity:** When `allOrNothing: true`, a single migration failure MUST cause the entire batch to roll back.
4. **Authentication:** Routes that do NOT export `export const config = { auth: false }` MUST have a valid `auth_context` or receive a `401` response.

---

## 5. Configuration Schema

```typescript
// medusa-config.ts (simplified)
export default defineConfig({
  projectConfig: {
    databaseUrl: string            // required, PostgreSQL DSN
    redisUrl?: string              // optional, for Redis-backed jobs/events
    http: {
      adminCors: string            // comma-separated origin allowlist
      storeCors: string
      authCors: string
      jwtSecret: string            // required
      cookieSecret: string         // required
    }
  },
  modules?: Record<string, ModuleDeclaration | false>,
  plugins?: PluginDescriptor[],
  featureFlags?: Record<string, boolean>,
  admin?: { backendUrl?: string; path?: string }
})
```

---

## 6. Error Codes

| `MedusaError.Types` | HTTP Status | Meaning |
|---|---|---|
| `NOT_FOUND` | 404 | Resource does not exist |
| `INVALID_DATA` | 400 | Malformed or logically invalid input |
| `NOT_ALLOWED` | 403 | Authorised but forbidden operation |
| `UNAUTHORIZED` | 401 | Missing or invalid credentials |
| `CONFLICT` | 409 | State conflict (e.g., duplicate key) |
| `DB_ERROR` | 500 | Database operation failed |
| `UNEXPECTED_STATE` | 500 | Internal invariant violated |

---

## 7. Compatibility Matrix

| Medusa Version | Node.js | PostgreSQL | MikroORM |
|---|---|---|---|
| 2.15.4 | ≥ 20 LTS | ≥ 14 | 6.x |
| 2.14.x | ≥ 18 LTS | ≥ 13 | 6.x |

---

## 8. Change Log (2.15.x)

| Version | Change |
|---|---|
| 2.15.4 | `reloadSingleModule()` added for HMR support |
| 2.15.0 | `additionalDataValidator` on `MiddlewareRoute` for per-route Zod schemas |
| 2.14.0 | `traceRoute` / `traceMiddleware` static hooks on `ApiLoader` |
