# Medusa Architecture Documentation

> **Medusa v2.15.4** — Building blocks for digital commerce  
> This documentation covers the complete architecture of the Medusa monorepo using arc42, SDD, SpecKit, UML, and ArchiMate notations.

---

## Contents

### Overall Architecture
| Document | Description |
|----------|-------------|
| [arc42.md](overall/arc42.md) | Full arc42 architecture document (all 12 sections) |
| [sdd.md](overall/sdd.md) | Software Design Document — system-level design |
| [speckit.md](overall/speckit.md) | SpecKit — functional and non-functional specifications |
| [diagrams/](overall/diagrams/) | System-level UML and ArchiMate diagrams |

### Packages (Core Framework)
| Package | Description |
|---------|-------------|
| [framework](packages/framework/) | Core runtime, HTTP layer, DI container, database |
| [core-flows](packages/core-flows/) | Pre-built reusable workflows |
| [workflows-sdk](packages/workflows-sdk/) | Workflow composition SDK |
| [types](packages/types/) | Shared TypeScript type definitions |
| [utils](packages/utils/) | Shared utilities and helpers |
| [modules-sdk](packages/modules-sdk/) | Module development SDK |
| [js-sdk](packages/js-sdk/) | JavaScript client SDK |

### Commerce Modules (35 modules)
| Module | Description |
|--------|-------------|
| [product](modules/product/) | Product catalog management |
| [order](modules/order/) | Order lifecycle management |
| [cart](modules/cart/) | Shopping cart |
| [payment](modules/payment/) | Payment processing |
| [customer](modules/customer/) | Customer accounts |
| [pricing](modules/pricing/) | Price lists and rules |
| [inventory](modules/inventory/) | Inventory tracking |
| [stock-location](modules/stock-location/) | Stock location management |
| [promotion](modules/promotion/) | Discounts and promotions |
| [fulfillment](modules/fulfillment/) | Fulfillment orchestration |
| [auth](modules/auth/) | Authentication and identity |
| [api-key](modules/api-key/) | API key management |
| [currency](modules/currency/) | Currency support |
| [region](modules/region/) | Regional configuration |
| [sales-channel](modules/sales-channel/) | Multi-channel sales |
| [tax](modules/tax/) | Tax calculation |
| [user](modules/user/) | Admin user management |
| [notification](modules/notification/) | Notification delivery |
| [file](modules/file/) | File storage abstraction |
| [event-bus-local](modules/event-bus-local/) | Local in-process event bus |
| [event-bus-redis](modules/event-bus-redis/) | Redis-backed event bus |
| [cache-inmemory](modules/cache-inmemory/) | In-memory cache |
| [cache-redis](modules/cache-redis/) | Redis-backed cache |
| [workflow-engine-inmemory](modules/workflow-engine-inmemory/) | In-process workflow engine |
| [workflow-engine-redis](modules/workflow-engine-redis/) | Redis-backed workflow engine |
| [locking](modules/locking/) | Distributed locking |
| [analytics](modules/analytics/) | Analytics event tracking |
| [rbac](modules/rbac/) | Role-based access control |
| [settings](modules/settings/) | System settings store |
| [store](modules/store/) | Store configuration |
| [index](modules/index/) | Search/index engine |
| [link-modules](modules/link-modules/) | Cross-module relationship links |

### Applications
| App | Description |
|-----|-------------|
| [backend](apps/backend/) | Medusa backend server (Express.js) |
| [storefront](apps/storefront/) | Next.js storefront reference |

### Providers
| Provider | Description |
|----------|-------------|
| [payment-stripe](providers/payment-stripe/) | Stripe payment provider |
| [auth-emailpass](providers/auth-emailpass/) | Email/password auth provider |
| [auth-github](providers/auth-github/) | GitHub OAuth provider |
| [auth-google](providers/auth-google/) | Google OAuth provider |
| [file-s3](providers/file-s3/) | AWS S3 file provider |
| [file-local](providers/file-local/) | Local filesystem provider |
| [fulfillment-manual](providers/fulfillment-manual/) | Manual fulfillment provider |
| [notification-sendgrid](providers/notification-sendgrid/) | SendGrid notification provider |
| [caching-redis](providers/caching-redis/) | Redis caching provider |

---

## Quick Navigation by Concern

### Data Flow
- System Context: [overall/diagrams/system-context.mmd](overall/diagrams/system-context.mmd)
- Module Data Flow: [overall/diagrams/data-flow.mmd](overall/diagrams/data-flow.mmd)

### Deployment
- Deployment View: [overall/diagrams/deployment.mmd](overall/diagrams/deployment.mmd)

### Module Relationships
- Module Dependency Graph: [overall/diagrams/module-dependencies.mmd](overall/diagrams/module-dependencies.mmd)
- ArchiMate Model: [overall/diagrams/archimate.md](overall/diagrams/archimate.md)

---

*Generated: 2026-05-31 | Medusa v2.15.4*
