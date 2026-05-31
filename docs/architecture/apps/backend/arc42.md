# @dtc/backend — arc42 Architecture Documentation

> Template version: 8.2 | Application: @dtc/backend | Medusa: 2.15.3

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@dtc/backend` is the server-side application for a headless DTC (Direct-to-Consumer) commerce platform. It must:

- Expose REST APIs consumed by the storefront and admin dashboard
- Manage the full commerce lifecycle: catalog, cart, checkout, orders, fulfilment
- Support loyalty points and rewards as a first-class domain concept
- Provide extensible admin tools for merchant operations
- Run reliably at scale with separated HTTP and background-worker processes

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Reliability** | API uptime ≥ 99.9 %; no data loss on partial failures |
| 2 | **Extensibility** | New modules and API routes deployable without core changes |
| 3 | **Performance** | Median API response < 150 ms under typical load |
| 4 | **Security** | CORS locked per namespace; JWT + cookie auth; secrets via env |
| 5 | **Observability** | Structured logs; traceable request IDs through workflow steps |

### 1.3 Stakeholders

| Role | Expectation |
|---|---|
| Platform Engineer | Clear module boundaries; deployable artefacts; migration safety |
| Backend Developer | Documented extension points; test harness |
| Merchant / Ops | Reliable admin dashboard; auditable order events |

---

## 2. Architecture Constraints

- **Runtime**: Node.js ≥ 20 (LTS)
- **Database**: PostgreSQL 14+ (MikroORM, schema managed by Medusa migrations)
- **Cache / Queue**: Redis (required for background jobs and event queue)
- **Framework**: Medusa v2 — no alternative IoC container or HTTP framework
- **Language**: TypeScript 5.6; compiled by `medusa build` using SWC
- **Deployment**: Container-first (Docker); single image, mode-switched via env var

---

## 3. System Scope and Context

### 3.1 Business Context

```
┌─────────────────────────────────────────────────────┐
│                   DTC Platform                      │
│                                                     │
│  ┌──────────────┐      ┌──────────────────────────┐ │
│  │  Storefront  │─────▶│    @dtc/backend           │ │
│  │  (Next.js)   │◀─────│    (Medusa Server)        │ │
│  └──────────────┘      │                          │ │
│                        │  ┌──────────────────────┐ │ │
│  ┌──────────────┐      │  │  Admin Dashboard     │ │ │
│  │  Admin UI    │─────▶│  │  (embedded React)    │ │ │
│  │  (Browser)   │◀─────│  └──────────────────────┘ │ │
│  └──────────────┘      └──────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 3.2 Technical Context

| External System | Protocol | Direction |
|---|---|---|
| PostgreSQL | TCP / pg wire | Backend → DB |
| Redis | TCP / RESP | Backend ↔ Redis |
| Stripe (via payment module) | HTTPS | Backend → Stripe |
| Email provider (via notification module) | HTTPS | Backend → Provider |
| Storefront | HTTPS REST | Storefront → Backend |
| Admin browser | HTTPS REST | Browser → Backend |

---

## 4. Solution Strategy

The application is structured as a **thin deployment shell around the Medusa framework**. Medusa provides 30+ built-in commerce modules (product, order, cart, customer, payment, fulfilment, inventory…). The project layer adds:

1. **Configuration** (`medusa-config.ts`) — wires built-in modules, credentials, CORS
2. **Custom modules** (`src/modules/`) — domain-specific extensions with hexagonal internals
3. **Module links** (`src/links/`) — cross-module foreign-key relationships
4. **Custom API routes** (`src/api/`) — project-specific HTTP endpoints
5. **Workflows** (`src/workflows/`) — multi-step saga-style business operations
6. **Subscribers** (`src/subscribers/`) — event-driven side effects
7. **Admin extensions** (`src/admin/`) — injected dashboard widgets

---

## 5. Building Block View

### Level 1 — System

```
@dtc/backend
├── HTTP Layer          Express router + Medusa middleware
├── Core Modules        30+ built-in Medusa commerce modules
├── Custom Modules      6 project-specific domain modules
├── Workflow Engine     @medusajs/framework/workflows-sdk
├── Event Bus           Redis-backed pub/sub
└── Database Layer      MikroORM → PostgreSQL
```

### Level 2 — Custom Modules

| Module | Primary Responsibility | Key Ports |
|---|---|---|
| `loyalty` | Points accrual, redemption, expiry | `ILoyaltyService`, `ILoyaltyRepository` |
| `admin-bff` | Aggregated admin queries | `IAdminBffService` |
| `commerce-catalog` | Extended catalog attributes | `ICatalogService` |
| `customer-identity` | Auth extensions, SSO | `IIdentityService` |
| `checkout-payment` | Payment session orchestration | `IPaymentOrchestrator` |
| `order-fulfillment` | Fulfilment routing & status | `IFulfillmentOrchestrator` |

---

## 6. Runtime View

### Sequence: Checkout → Order Placed → Loyalty Award

```
Storefront          Backend HTTP          Workflow Engine       Loyalty Module
    │                    │                      │                    │
    │── POST /store/carts/:id/complete ─────────▶│                    │
    │                    │── completeCartWorkflow.run() ─────────────▶│
    │                    │                      │── validateCart()    │
    │                    │                      │── chargePayment()   │
    │                    │                      │── createOrder()     │
    │                    │                      │── emit order.placed │
    │                    │                      │                    │
    │                    │◀── { order } ─────────│                    │
    │◀── 200 { order } ──│                      │                    │
    │                    │                      │                    │
    │          [async]   │◀── subscriber: order.placed ──────────────│
    │                    │── awardLoyaltyPointsWorkflow.run() ───────▶│
    │                    │                      │── calculatePoints() │
    │                    │                      │── persistAward() ──▶│
```

---

## 7. Deployment View

```
┌─────────────────────────────────────────────┐
│  Container: backend-server                  │
│  WORKER_MODE=server                         │
│  Replicas: 2–N  (horizontal scale)          │
│                                             │
│  Express HTTP :9000                         │
│  → Admin API  /admin/*                      │
│  → Store API  /store/*                      │
│  → Auth API   /auth/*                       │
└───────────────────┬─────────────────────────┘
                    │ Redis queue
┌───────────────────▼─────────────────────────┐
│  Container: backend-worker                  │
│  WORKER_MODE=worker                         │
│  Replicas: 1–N                              │
│                                             │
│  Subscriber runner                          │
│  Scheduled job runner                       │
└───────────────────┬─────────────────────────┘
                    │
         ┌──────────▼──────────┐
         │     PostgreSQL      │
         │     Redis           │
         └─────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### 8.1 Error Handling

All errors thrown inside service and workflow step code use `MedusaError(type, message)`. The HTTP layer maps error types to status codes (404, 400, 422, 500). Workflow compensation functions are called automatically on step failure, rolling back side effects.

### 8.2 Security

- CORS is enforced at the framework level per API namespace.
- Admin routes require a valid JWT in the `Authorization: Bearer` header.
- Store routes require a valid `x-publishable-api-key` header.
- Secrets (`JWT_SECRET`, `COOKIE_SECRET`) are injected via environment variables; never hardcoded.

### 8.3 Testing Strategy

| Level | Tool | Scope |
|---|---|---|
| Unit | Jest + SWC | Module services, workflow steps in isolation |
| Integration (modules) | `@medusajs/test-utils` | Full module lifecycle with real DB |
| Integration (HTTP) | Jest + `supertest` | Full API request/response including auth |

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | Hexagonal pattern for custom modules | Enables unit-testing ports without DB; swappable adapters |
| AD-02 | Worker mode separation | Allows HTTP and background tiers to scale independently |
| AD-03 | Module links for cross-domain relations | Decouples modules; avoids tight coupling via foreign keys |
| AD-04 | Server Actions avoided in backend | Backend is pure REST; no Next.js coupling |

---

## 10. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Medusa minor-version upgrades may change module APIs | Medium | Pin exact versions; review changelog before upgrade |
| Custom module schema divergence during Medusa core upgrades | High | Run integration tests against upgraded Medusa before deploying |
| Redis single point of failure | High | Use Redis Sentinel or Cluster in production |
