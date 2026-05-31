# arc42 Architecture Document — `@medusajs/framework`

**Version:** 2.15.4  
**Template:** arc42 v8.2  

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/framework` must:

- Bootstrap a Medusa commerce application from a single `medusa-config.ts` entry point.
- Provide a type-safe HTTP layer with filesystem-based route discovery.
- Manage the full lifecycle of all commerce modules inside one process.
- Offer a deterministic, rollback-safe database migration pipeline.
- Allow zero-config background jobs and event subscribers.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Extensibility** | A developer adds a custom route by placing a file — no registration code required. |
| 2 | **Reliability** | A module crash is isolated; other modules continue to serve requests. |
| 3 | **Developer Experience** | Type errors in route handlers surface at compile time. |
| 4 | **Performance** | Cold-start under 3 seconds for a default installation. |

### 1.3 Stakeholders

| Role | Expectation |
|---|---|
| Application Developer | Easy-to-follow file conventions, excellent TypeScript DX |
| Module Author | Stable container API, clear lifecycle hooks |
| Platform Operator | Reliable DB migrations, observable telemetry |
| Core Contributor | Testable, loosely coupled sub-systems |

---

## 2. Architecture Constraints

- **Express.js** is the HTTP transport. Fastify or other adapters are out of scope.
- **Awilix** is the IoC container. Replacing it is a breaking change.
- **MikroORM + PostgreSQL** is the only officially supported database stack.
- All sub-systems must work in a single Node.js process (no inter-process communication required).

---

## 3. System Context

```
┌─────────────────────────────────────────────────────┐
│                   Node.js Process                   │
│                                                     │
│  ┌─────────────┐   HTTP    ┌───────────────────┐   │
│  │   Client    │◄─────────►│  @medusajs/        │   │
│  │ (Browser /  │           │  framework         │   │
│  │  Mobile /   │           │  (Express server)  │   │
│  │  CLI)       │           └────────┬──────────┘   │
│  └─────────────┘                    │               │
│                              DI Container            │
│                         ┌──────────┴───────────┐    │
│                         │   Commerce Modules   │    │
│                         │  (product, order…)   │    │
│                         └──────────┬───────────┘    │
│                                    │                 │
│                              PostgreSQL              │
└─────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

| Problem | Solution |
|---|---|
| Dozens of modules to load | `MedusaAppLoader` merges defaults and runs loaders in sequence |
| Route management at scale | Filesystem convention; no central router registry |
| Module isolation | Each module owns its MikroORM entities and migrations |
| Cross-module data | `RemoteLink` + `Query` abstraction, not direct DB joins |
| Request safety | Request-scoped Awilix container fork per Express request |

---

## 5. Building Blocks (Level 1)

```
@medusajs/framework
├── MedusaAppLoader       — root orchestrator
├── HTTP (ApiLoader)      — Express + route filesystem
├── Container             — Awilix IoC container
├── Database              — Knex pool + MikroORM
├── Config                — medusa-config.ts parser
├── Modules SDK           — module loader & registry
├── Jobs                  — cron job file loader
├── Subscribers           — event subscriber file loader
├── Policies              — auth / CORS / rate-limit
├── Telemetry             — optional usage reporting
├── Links                 — cross-module linker
└── Zod                   — request validation utilities
```

---

## 6. Building Blocks (Level 2) — HTTP Sub-system

```
ApiLoader
│
├── routes-finder
│     └── Recursive FS walk → string[]  (absolute paths to route.ts files)
│
├── routes-sorter
│     └── Sort by: static > param > wildcard > catch-all
│
├── routes-loader
│     ├── Dynamic import(filePath)
│     ├── Extract named exports (GET, POST, …)
│     └── app.METHOD(matcher, handler)
│
└── middleware-file-loader
      ├── Import middlewares.ts
      └── Register MiddlewareRoute[] entries on Express
```

---

## 7. Runtime View

### 7.1 Application Startup Sequence

```
1. new MedusaAppLoader(options)
2.   ConfigLoader.load()        → ConfigModule
3.   ContainerBuilder.build()   → MedusaContainer
4.   DatabaseLoader.connect()   → Knex pool injected into container
5.   ModulesLoader.run()        → all module services registered
6.   LinkModulesLoader.run()    → cross-module links resolved
7.   ApiLoader.load()           → HTTP routes registered on Express
8.   JobsLoader.schedule()      → cron jobs registered
9.   SubscribersLoader.bind()   → event listeners attached
10.  app.listen(port)           → server ready
```

### 7.2 Incoming HTTP Request

```
Request arrives
  → CORS policy middleware
  → Rate-limit policy middleware
  → Authentication policy middleware (JWT / API key)
  → Body parser (JSON / multipart)
  → Zod validate-body (if schema defined)
  → Route handler (req.scope resolves DI services)
  → Response
```

---

## 8. Deployment View

`@medusajs/framework` runs inside a single Node.js process. It has no mandatory external runtime dependencies beyond PostgreSQL. In production deployments:

- **Horizontal scaling**: Multiple instances share the same PostgreSQL database; the Knex pool is per-process.
- **Background jobs**: Each instance runs its own job scheduler. For distributed job deduplication, operators should use an external lock (e.g., Redis advisory locks).
- **Migrations**: Run once before deployment via `npx medusa db:migrate`.

---

## 9. Cross-Cutting Concerns

### 9.1 Logging

The `logger` service (registered in the container) emits structured JSON logs. Log level is configurable via `projectConfig.logLevel`. All framework sub-systems accept the logger via DI.

### 9.2 Observability

`ApiLoader.traceRoute` and `ApiLoader.traceMiddleware` are static optional hooks. When set (e.g., by an OpenTelemetry plugin), every route and middleware invocation is automatically wrapped in a span.

### 9.3 Error Propagation

Framework errors use `MedusaError` with typed error kinds. The Express global error handler serialises these into RFC 7807 Problem Details-compatible JSON responses.

### 9.4 Configuration Validation

`medusa-config.ts` is validated against an internal Zod schema on load. Missing required fields (e.g., `databaseUrl`) throw immediately at startup rather than at first use.

---

## 10. Architecture Decisions

### ADR-001: Filesystem-based Route Discovery

**Context:** Managing a growing number of API routes across plugins and custom code.  
**Decision:** Use filesystem scanning instead of a central route registry.  
**Consequences:** Routes are co-located with their logic; adding a route requires no config change. Trade-off: startup time scales with file count (acceptable at current scale).

### ADR-002: Shared Knex Pool

**Context:** 30+ modules each potentially opening their own DB connections.  
**Decision:** One shared Knex pool injected into all modules.  
**Consequences:** Predictable connection count; modules cannot tune pool independently.

### ADR-003: Request-Scoped Container Fork

**Context:** Preventing data leakage between concurrent requests.  
**Decision:** Fork the Awilix container per request for scoped registrations.  
**Consequences:** Clean isolation; slight allocation overhead per request (negligible vs. DB round-trips).
