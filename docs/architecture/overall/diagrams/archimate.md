# ArchiMate 3.1 Architecture Description — Medusa v2.15.4

> **Notation**: This document uses ArchiMate 3.1 text representation.  
> ArchiMate elements are expressed as structured tables per layer, followed by relationship tables.  
> Stereotype notation: `«type»` Name — Description.

---

## Table of Contents

1. [Motivation Layer](#1-motivation-layer)
2. [Strategy Layer](#2-strategy-layer)
3. [Business Layer](#3-business-layer)
4. [Application Layer](#4-application-layer)
5. [Technology Layer](#5-technology-layer)
6. [Physical Layer](#6-physical-layer)
7. [Cross-Layer Relationships](#7-cross-layer-relationships)
8. [Key Viewpoints](#8-key-viewpoints)

---

## 1. Motivation Layer

### 1.1 Drivers

| Element Type | Name | Description |
|---|---|---|
| `Driver` | Commerce Fragmentation | Merchants must integrate multiple disjointed commerce tools (payments, CMS, ERP, PIM), raising cost and complexity. |
| `Driver` | Vendor Lock-in | Proprietary SaaS commerce platforms restrict customization and data portability. |
| `Driver` | Developer Experience Deficit | Existing headless solutions lack strong typing, transactional safety, and modularity for fast iteration. |
| `Driver` | Global Commerce Complexity | Multi-currency, multi-region, multi-channel commerce is difficult to orchestrate consistently. |

### 1.2 Assessments

| Element Type | Name | Description |
|---|---|---|
| `Assessment` | Open-Source Advantage | Open-source licensing eliminates vendor lock-in and allows full codebase transparency. |
| `Assessment` | Modular Architecture Value | Independent modules reduce coupling, allowing teams to replace or extend individual capabilities. |
| `Assessment` | TypeScript Type Safety | End-to-end TypeScript with strict typing reduces runtime errors and improves IDE productivity. |

### 1.3 Goals

| Element Type | Name | Description |
|---|---|---|
| `Goal` | Composable Commerce | Enable merchants to compose custom commerce stacks from independent, swappable modules. |
| `Goal` | Developer Productivity | Reduce time-to-production for custom commerce features to hours, not weeks. |
| `Goal` | Operational Resilience | Ensure transactional consistency and automatic rollback for all commerce operations via workflow sagas. |
| `Goal` | Multi-Region Commerce | Support multiple regions, currencies, tax configurations, and sales channels natively. |
| `Goal` | Extensibility Without Forking | Allow merchants to extend behaviour (custom modules, API routes, workflows) without modifying core. |

### 1.4 Principles

| Element Type | Name | Description |
|---|---|---|
| `Principle` | Module Independence | Each commerce module owns its data model, service interface, and repository. No direct cross-module DB joins. |
| `Principle` | Workflow-First Mutations | All state-changing operations are orchestrated via the Workflow Engine, ensuring compensability. |
| `Principle` | Dependency Injection | All services and modules are resolved through the Awilix DI container — no static singletons. |
| `Principle` | Event-Driven Side Effects | Cross-module side effects are triggered via domain events, not direct service calls. |
| `Principle` | API-First Design | Every capability is exposed via REST API with strong request/response typing (Zod + TypeScript). |
| `Principle` | Convention over Configuration | Sensible defaults with explicit overrides; zero-config startup for local development. |

---

## 2. Strategy Layer

### 2.1 Capabilities

| Element Type | Name | Description |
|---|---|---|
| `Capability` | Catalog Management | Create, update, and publish products with variants, options, collections, categories, and tags. |
| `Capability` | Order Lifecycle Management | Process orders from cart creation through payment, fulfillment, and return/refund. |
| `Capability` | Payment Orchestration | Accept, authorize, capture, refund, and void payments via pluggable provider pattern. |
| `Capability` | Inventory Management | Track stock levels across multiple locations with reservations and availability checks. |
| `Capability` | Promotions & Pricing | Define price lists, discount codes, automatic promotions, and region-specific pricing. |
| `Capability` | Customer Identity | Register, authenticate, and manage customer accounts across multiple sales channels. |
| `Capability` | Multi-Region Operations | Configure regions with currencies, tax providers, fulfillment options, and payment methods. |
| `Capability` | Fulfillment & Shipping | Manage fulfillment providers, shipping options, and delivery tracking. |
| `Capability` | Extensibility Platform | Register custom modules, API routes, workflows, and scheduled jobs without core modification. |

### 2.2 Resources

| Element Type | Name | Description |
|---|---|---|
| `Resource` | Medusa Backend Codebase | TypeScript monorepo (~30 packages) under Apache 2.0 license. |
| `Resource` | PostgreSQL Database | Relational store for all persistent commerce data. |
| `Resource` | Redis Instance | In-memory store for caching, event bus transport, and job queues. |
| `Resource` | Module Ecosystem | 30+ official modules and 15+ provider implementations. |
| `Resource` | Developer Community | Open-source contributors, plugin authors, and community marketplace. |

---

## 3. Business Layer

### 3.1 Business Actors

| Element Type | Name | Description |
|---|---|---|
| `BusinessActor` | Merchant | Operates the commerce store. Manages catalog, processes orders, configures settings, and monitors analytics via the Admin Dashboard. |
| `BusinessActor` | Customer | End consumer. Browses the catalog, manages their cart, completes checkout, and tracks orders via the Storefront. |
| `BusinessActor` | Developer | Technical integrator. Builds storefronts, custom plugins, and integrations using the Medusa REST API and SDK. |
| `BusinessActor` | Payment Provider | External financial institution (Stripe, PayPal, Adyen) that authorizes and processes payment transactions. |
| `BusinessActor` | Fulfillment Provider | Shipping/logistics partner (FedEx, DHL, custom) that handles physical delivery of orders. |
| `BusinessActor` | Email Provider | Transactional email service (SendGrid, Resend, Postmark) for customer notifications. |

### 3.2 Business Roles

| Element Type | Name | Description |
|---|---|---|
| `BusinessRole` | Store Administrator | Merchant role with full access to admin operations. |
| `BusinessRole` | Store Operator | Merchant role with restricted access (orders/fulfillment only). |
| `BusinessRole` | Registered Customer | Authenticated customer with order history and saved addresses. |
| `BusinessRole` | Guest Customer | Unauthenticated customer; checkout via email. |
| `BusinessRole` | API Integrator | Developer role consuming public or admin REST APIs. |

### 3.3 Business Services

| Element Type | Name | Description |
|---|---|---|
| `BusinessService` | Product Catalog Service | Expose product listings, variants, availability, and pricing to customers and storefronts. |
| `BusinessService` | Order Processing Service | Accept customer orders, process payments, trigger fulfillment, and manage post-purchase operations. |
| `BusinessService` | Customer Account Service | Register customers, manage profiles, addresses, and order history. |
| `BusinessService` | Promotions Service | Evaluate and apply discount codes, automatic promotions, and gift cards at checkout. |
| `BusinessService` | Notification Service | Send transactional emails and push notifications for order status changes and account events. |
| `BusinessService` | Reporting Service | Provide sales analytics, revenue reports, and operational metrics to merchants. |

### 3.4 Business Processes

| Element Type | Name | Description |
|---|---|---|
| `BusinessProcess` | Place Order | Customer adds items to cart → applies promotions → enters shipping/payment → places order → receives confirmation. |
| `BusinessProcess` | Manage Catalog | Merchant creates products → defines variants/options → sets pricing → publishes to sales channels. |
| `BusinessProcess` | Process Payment | System authorizes payment at order creation → captures on fulfillment → issues refund on return. |
| `BusinessProcess` | Fulfill Order | Warehouse receives order → picks and packs items → ships with carrier → updates tracking. |
| `BusinessProcess` | Handle Return/Refund | Customer requests return → merchant approves → inventory restocked → payment refunded. |
| `BusinessProcess` | Manage Promotions | Merchant creates promotion rules → sets conditions/actions → activates/deactivates campaign. |
| `BusinessProcess` | Customer Registration | Customer provides email/password or OAuth → account created → welcome email sent. |

### 3.5 Business Objects

| Element Type | Name | Description |
|---|---|---|
| `BusinessObject` | Order | Confirmed purchase record with line items, totals, payment, and fulfillment state. |
| `BusinessObject` | Cart | Pre-order container of line items with applied promotions and checkout context. |
| `BusinessObject` | Product | Saleable item with variants, options, images, and metadata. |
| `BusinessObject` | Customer | Registered or guest buyer with contact and address information. |
| `BusinessObject` | Payment | Financial transaction record linked to an order, with provider reference and state machine. |
| `BusinessObject` | Fulfillment | Shipment record tracking the delivery of order items. |
| `BusinessObject` | Promotion | Discount rule with conditions (cart subtotal, customer group) and actions (percentage off, fixed amount). |
| `BusinessObject` | Price List | Set of variant prices that override base prices for specific conditions (regions, customer groups). |

---

## 4. Application Layer

### 4.1 Application Components

| Element Type | Name | Description |
|---|---|---|
| `ApplicationComponent` | Admin Dashboard | React SPA (Vite + React Router). Back-office UI at `packages/admin/dashboard`. Communicates with Admin API over REST/HTTPS. |
| `ApplicationComponent` | Storefront | Next.js application (App Router). Customer-facing shop. Connects to Store API. Reference implementation at `starters/`. |
| `ApplicationComponent` | Medusa API Server | Node.js / Express application. Entry point: `packages/medusa`. Exposes Admin, Store, and Custom API routes. |
| `ApplicationComponent` | Workflow Engine | Saga orchestrator at `packages/core/workflows-sdk`. Executes typed steps with compensation, parallelism, and lifecycle hooks. |
| `ApplicationComponent` | DI Container | Awilix IoC container at `packages/core/framework/src/container`. Registers and resolves all services and modules. |
| `ApplicationComponent` | Event Bus Module | Domain event publisher/subscriber. Redis transport (`@medusajs/event-bus-redis`) or in-process (`@medusajs/event-bus-local`). |
| `ApplicationComponent` | Cache Module | Key-value cache abstraction. Redis backend (`@medusajs/cache-redis`) or in-memory (`@medusajs/cache-inmemory`). |
| `ApplicationComponent` | Product Module | `@medusajs/product`. Manages products, variants, options, images, collections, categories, tags. |
| `ApplicationComponent` | Order Module | `@medusajs/order`. Manages order lifecycle, line items, adjustments, shipping methods, returns, claims. |
| `ApplicationComponent` | Cart Module | `@medusajs/cart`. Manages cart creation, item management, address, shipping, and payment session setup. |
| `ApplicationComponent` | Payment Module | `@medusajs/payment`. Provider-agnostic payment session management, capture, refund, and webhook handling. |
| `ApplicationComponent` | Customer Module | `@medusajs/customer`. Customer accounts, addresses, customer groups, and group memberships. |
| `ApplicationComponent` | Pricing Module | `@medusajs/pricing`. Price sets, price lists, price rules, and money amount management. |
| `ApplicationComponent` | Inventory Module | `@medusajs/inventory`. Inventory items, levels per location, reservations, and availability computation. |
| `ApplicationComponent` | Fulfillment Module | `@medusajs/fulfillment`. Fulfillment sets, shipping options, fulfillment providers, and delivery tracking. |
| `ApplicationComponent` | Promotion Module | `@medusajs/promotion`. Promotion rules, conditions, actions, campaign management, and application logic. |
| `ApplicationComponent` | Auth Module | `@medusajs/auth`. Authentication provider abstraction (email/password, Google OAuth, GitHub OAuth). |
| `ApplicationComponent` | Region Module | `@medusajs/region`. Region configuration with currencies, countries, and tax provider assignment. |
| `ApplicationComponent` | Sales Channel Module | `@medusajs/sales-channel`. Logical store channels (web, mobile, POS) with product availability scoping. |
| `ApplicationComponent` | Stock Location Module | `@medusajs/stock-location`. Physical warehouse and fulfillment location management. |
| `ApplicationComponent` | Notification Module | `@medusajs/notification`. Notification provider abstraction for email, SMS, and push. |
| `ApplicationComponent` | File Module | `@medusajs/file`. Asset upload/download abstraction (S3, local disk). |
| `ApplicationComponent` | Workflow Store Module | `@medusajs/workflow-engine-inmemory` / `redis`. Persists workflow execution state for long-running sagas. |

### 4.2 Application Services (API Interfaces)

| Element Type | Name | Description |
|---|---|---|
| `ApplicationService` | Admin REST API | `/admin/*` routes. Authenticated (JWT). Full CRUD for all commerce resources. |
| `ApplicationService` | Store REST API | `/store/*` routes. Public + session-auth. Customer-facing operations: product listing, cart, checkout, account. |
| `ApplicationService` | Custom Routes API | Merchant-defined routes registered in `medusa-config.ts`. Extend the API surface without core changes. |
| `ApplicationService` | Webhook Receiver | `/webhooks/*` routes. Receive and process payment provider and external system webhooks. |

### 4.3 Application Interfaces

| Element Type | Name | Description |
|---|---|---|
| `ApplicationInterface` | IProductModuleService | TypeScript interface defining all product module operations. Implemented by `ProductModuleService`. |
| `ApplicationInterface` | IOrderModuleService | TypeScript interface for order lifecycle operations. |
| `ApplicationInterface` | IPaymentModuleService | TypeScript interface for payment session management and provider delegation. |
| `ApplicationInterface` | IPaymentProvider | Provider plugin interface. Implemented by Stripe, PayPal, etc. |
| `ApplicationInterface` | IAuthProvider | Authentication provider interface. Email/password, OAuth implementations. |
| `ApplicationInterface` | IFileProvider | File storage provider interface. S3, local, R2 implementations. |
| `ApplicationInterface` | INotificationProvider | Notification delivery interface. SendGrid, Resend, Postmark implementations. |
| `ApplicationInterface` | IFulfillmentProvider | Fulfillment provider interface. FedEx, DHL, manual implementations. |

### 4.4 Application Interactions (Core Flows)

| Element Type | Name | Description |
|---|---|---|
| `ApplicationInteraction` | createCartWorkflow | Creates cart with items, customer, sales channel, shipping, and payment context. |
| `ApplicationInteraction` | completeCartWorkflow | Completes checkout: validates cart → creates order → initiates payment → confirms. |
| `ApplicationInteraction` | createOrderWorkflow | Creates confirmed order from completed cart with reservations. |
| `ApplicationInteraction` | capturePaymentWorkflow | Captures authorized payment on order fulfillment. |
| `ApplicationInteraction` | createFulfillmentWorkflow | Creates fulfillment, deducts inventory, triggers shipping provider. |
| `ApplicationInteraction` | createReturnWorkflow | Processes return request, restocks inventory, and triggers refund. |
| `ApplicationInteraction` | deletePromotionsWorkflow | Soft-deletes promotions with compensation (restore on failure). |
| `ApplicationInteraction` | updateProductVariantsWorkflow | Updates variant data and associated pricing in a single saga. |

---

## 5. Technology Layer

### 5.1 Technology Nodes

| Element Type | Name | Description |
|---|---|---|
| `Node` | Node.js 20 Runtime | JavaScript/TypeScript runtime (LTS). Hosts the API server and worker processes. |
| `Node` | PostgreSQL 15 Server | Primary relational database server. Stores all persistent commerce data. |
| `Node` | Redis 7 Server | In-memory data store. Dual role: event-bus transport and application cache layer. |
| `Node` | Docker Container Host | OCI-compatible container runtime (Docker Engine / Podman). Hosts all application containers. |
| `Node` | Nginx Reverse Proxy | HTTP/HTTPS reverse proxy with TLS termination. Routes traffic to backend containers. |
| `Node` | S3-Compatible Object Store | Blob storage for product images and uploaded assets (AWS S3, MinIO, Cloudflare R2). |
| `Node` | CDN (optional) | Content Delivery Network for static asset distribution and edge caching. |

### 5.2 System Software

| Element Type | Name | Description |
|---|---|---|
| `SystemSoftware` | Express.js 4 | Node.js HTTP framework. Handles routing, middleware pipeline, and request/response lifecycle. |
| `SystemSoftware` | MikroORM 6 | TypeScript-first ORM. Entity management, migrations, unit of work pattern, and query builder. |
| `SystemSoftware` | Awilix | Dependency injection container for Node.js. Manages service lifetimes and resolution. |
| `SystemSoftware` | BullMQ | Redis-backed job queue. Manages background job execution and scheduling. |
| `SystemSoftware` | Zod | TypeScript-first schema validation. Validates and types all API request bodies. |
| `SystemSoftware` | ioredis | Redis client for Node.js. Used by event bus, cache, and BullMQ. |
| `SystemSoftware` | Winston | Structured logging library. Request logs, error logs, and audit trails. |
| `SystemSoftware` | Vite | Frontend build tool for Admin Dashboard SPA. |
| `SystemSoftware` | Turbo (Turborepo) | Monorepo build orchestration with caching and parallel execution. |

### 5.3 Technology Services

| Element Type | Name | Description |
|---|---|---|
| `TechnologyService` | HTTP Service | Express server bound to port 9000. Serves REST API requests. |
| `TechnologyService` | Database Connection Pool | MikroORM EntityManager pool. Maintains persistent connections to PostgreSQL. |
| `TechnologyService` | Event Bus Service | Redis pub/sub channel management. Routes domain events to registered subscribers. |
| `TechnologyService` | Cache Service | Redis GET/SET/DEL operations. TTL-based cache invalidation. |
| `TechnologyService` | Job Queue Service | BullMQ queue management. Delayed jobs, retries, and dead-letter handling. |
| `TechnologyService` | File Storage Service | S3 presigned URL generation, multipart upload, and object retrieval. |
| `TechnologyService` | Migration Service | MikroORM schema migration runner. Applied at startup or via CLI. |

### 5.4 Communication Paths

| Element Type | Name | Description |
|---|---|---|
| `CommunicationPath` | Admin API Path | HTTPS :443 → Nginx → HTTP :9000 → Express Router → Admin handlers. |
| `CommunicationPath` | Store API Path | HTTPS :443 → Nginx → HTTP :9000 → Express Router → Store handlers. |
| `CommunicationPath` | DB Connection Path | TCP :5432 — MikroORM ↔ PostgreSQL. Connection pooled. |
| `CommunicationPath` | Redis Connection Path | TCP :6379 — ioredis ↔ Redis. Persistent connection with reconnect. |
| `CommunicationPath` | Payment Webhook Path | HTTPS from Stripe/PayPal → `/webhooks/payment/*` → PaymentModule. |
| `CommunicationPath` | Worker Bus Path | BullMQ (Redis) — API process enqueues jobs → Worker process consumes. |

---

## 6. Physical Layer

### 6.1 Equipment & Facilities

| Element Type | Name | Description |
|---|---|---|
| `Equipment` | Application Server | Server or VM hosting Docker containers. Minimum: 2 vCPU, 4 GB RAM for production. |
| `Equipment` | Database Server | Dedicated PostgreSQL host (or managed service: RDS, Cloud SQL, Neon). |
| `Equipment` | Redis Server | Dedicated Redis host (or managed service: Elasticache, Redis Cloud, Upstash). |
| `Equipment` | Object Storage | AWS S3, Cloudflare R2, or MinIO installation for asset storage. |
| `Facility` | Primary Data Center | Production hosting environment (AWS, GCP, Azure, Hetzner, etc.). |
| `Facility` | Disaster Recovery Site | Backup site with PostgreSQL replica and Redis standby (optional). |

### 6.2 Distribution Networks

| Element Type | Name | Description |
|---|---|---|
| `DistributionNetwork` | Public Internet | HTTPS traffic from browsers and mobile clients to Nginx. |
| `DistributionNetwork` | Private Docker Network | Internal bridge network (`medusa-network`). Containers communicate by service name. |
| `DistributionNetwork` | CDN Edge Network | Cloudflare or CloudFront edge PoPs caching static assets. |

---

## 7. Cross-Layer Relationships

### 7.1 Realisation Relationships

| From (Application) | Relationship | To (Business) |
|---|---|---|
| Admin REST API | `Realises` | Store Administrator business role |
| Store REST API | `Realises` | Registered Customer / Guest Customer business roles |
| createCartWorkflow | `Realises` | Place Order business process |
| createFulfillmentWorkflow | `Realises` | Fulfill Order business process |
| capturePaymentWorkflow | `Realises` | Process Payment business process |
| createReturnWorkflow | `Realises` | Handle Return/Refund business process |
| Product Module | `Realises` | Product Catalog Service |
| Promotion Module | `Realises` | Promotions Service |
| Notification Module | `Realises` | Notification Service |

### 7.2 Serving Relationships (Technology → Application)

| From (Technology) | Relationship | To (Application) |
|---|---|---|
| Node.js 20 Runtime | `Serves` | Medusa API Server |
| Express.js 4 | `Serves` | Admin REST API, Store REST API |
| MikroORM 6 | `Serves` | All Commerce Modules (data persistence) |
| Awilix | `Serves` | DI Container (module resolution) |
| BullMQ | `Serves` | Workflow Engine (async step execution) |
| Redis 7 Server | `Serves` | Event Bus Module, Cache Module, Job Queue |
| PostgreSQL 15 Server | `Serves` | All persistence via MikroORM |
| S3-Compatible Store | `Serves` | File Module |

### 7.3 Association Relationships (Module Dependencies)

| From (Module) | Association | To (Module) | Note |
|---|---|---|---|
| Cart | `Associated with` | Product | Line items reference product variants |
| Cart | `Associated with` | Customer | Cart owner identity |
| Cart | `Associated with` | Pricing | Price calculation for line items |
| Cart | `Associated with` | Promotion | Discount code application |
| Cart | `Associated with` | Region | Tax and currency context |
| Cart | `Associated with` | Sales Channel | Product availability scope |
| Order | `Created from` | Cart | Completed cart becomes an order |
| Order | `Associated with` | Payment | Payment session and collection |
| Order | `Associated with` | Fulfillment | Shipment tracking |
| Order | `Associated with` | Inventory | Stock reservation on order placement |
| Pricing | `Associated with` | Product | Prices attached to variants |
| Pricing | `Associated with` | Region | Regional price overrides |
| Promotion | `Associated with` | Pricing | Promotion applies price adjustments |
| Fulfillment | `Associated with` | Inventory | Stock deduction on fulfillment |
| Fulfillment | `Associated with` | Stock Location | Warehouse sourcing |
| Auth | `Associated with` | User | Admin identity management |
| Auth | `Associated with` | Customer | Shopper identity management |

---

## 8. Key Viewpoints

### 8.1 Layered Viewpoint (Summary)

```
┌─────────────────────────────────────────────────────────────────┐
│  MOTIVATION LAYER                                               │
│  Driver: Commerce Fragmentation → Goal: Composable Commerce    │
│  Principle: Module Independence, Workflow-First, Event-Driven  │
├─────────────────────────────────────────────────────────────────┤
│  BUSINESS LAYER                                                 │
│  Actors: Merchant, Customer, Developer                          │
│  Processes: Place Order, Fulfill, Payment, Return, Catalog Mgmt │
├─────────────────────────────────────────────────────────────────┤
│  APPLICATION LAYER                                              │
│  Components: API Server, Workflow Engine, 30+ Modules           │
│  Interfaces: IProductModuleService, IPaymentProvider, …         │
│  Flows: createCartWorkflow, completeCartWorkflow, …             │
├─────────────────────────────────────────────────────────────────┤
│  TECHNOLOGY LAYER                                               │
│  Runtime: Node.js 20, Express, MikroORM, Awilix, BullMQ        │
│  Services: HTTP :9000, DB Pool (PG), Pub/Sub (Redis)           │
├─────────────────────────────────────────────────────────────────┤
│  PHYSICAL LAYER                                                 │
│  Equipment: App Server, PostgreSQL Host, Redis Host, S3         │
│  Network: Docker bridge, Public HTTPS, CDN                      │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Application Cooperation Viewpoint

The following describes the primary application cooperation for the **Place Order** use case:

```
Customer Browser
  └─► Storefront (Next.js)
        └─► Store API (/store/carts, /store/orders)
              └─► Middleware (auth + validation)
                    └─► completeCartWorkflow
                          ├─► Cart Module    (validate + retrieve cart)
                          ├─► Pricing Module (calculate totals)
                          ├─► Promotion Module (apply discounts)
                          ├─► Order Module   (create order record)
                          ├─► Payment Module (create payment session)
                          ├─► Inventory Module (create reservations)
                          └─► Notification Module (send confirmation)
```

### 8.3 Technology Usage Viewpoint

```
Medusa API Server
  ├── uses Express.js     → HTTP routing and middleware
  ├── uses Awilix         → DI resolution of all modules
  ├── uses MikroORM       → ORM ↔ PostgreSQL
  ├── uses ioredis        → Event Bus pub/sub ↔ Redis
  ├── uses ioredis        → Cache reads/writes ↔ Redis
  └── uses BullMQ         → Background jobs ↔ Redis

Worker Process
  ├── uses BullMQ Worker  → Job consumption from Redis
  └── uses Event Subscribers → Async domain event handling
```

---

*Document generated for Medusa v2.15.4. ArchiMate 3.1 © The Open Group.*
