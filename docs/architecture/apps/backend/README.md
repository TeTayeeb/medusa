# @dtc/backend — Medusa Backend Application

> Medusa v2.15.3 · Node.js ≥ 20 · TypeScript 5.6

## Overview

`@dtc/backend` is the deployable server-side application for the DTC commerce platform. It is built on **Medusa v2**, a modular, headless commerce framework. The backend exposes REST APIs for the storefront and admin dashboard, orchestrates business logic through workflows, and hosts a set of domain-specific custom modules that extend Medusa's built-in capabilities.

The application ships as a single Node.js process that can run in one of three **worker modes**:

| `WORKER_MODE` | Role |
|---|---|
| `server` | Handles HTTP requests only |
| `worker` | Processes background jobs and subscribers only |
| `shared` *(default)* | Runs both HTTP and background processing in one process |

---

## Project Structure

```
apps/backend/
├── medusa-config.ts          # Project configuration (modules, DB, CORS, HTTP)
├── src/
│   ├── admin/                # Admin dashboard extensions
│   │   ├── i18n/             # Internationalisation strings
│   │   └── widgets/          # Custom admin widgets (injected into dashboard)
│   ├── api/                  # Project-specific API routes
│   │   ├── admin/custom/     # Custom admin endpoints
│   │   ├── store/custom/     # Custom storefront endpoints
│   │   └── cloud/auth/       # Cloud-specific auth endpoints
│   ├── jobs/                 # Scheduled background jobs
│   ├── links/                # Module link definitions
│   ├── migration-scripts/    # Data migration and seed scripts
│   │   └── initial-data-seed.ts
│   ├── modules/              # Custom domain modules (ports/adapters pattern)
│   │   ├── admin-bff/        # Admin back-end-for-frontend
│   │   ├── checkout-payment/ # Payment orchestration
│   │   ├── commerce-catalog/ # Catalog extensions
│   │   ├── customer-identity/# Customer auth & identity
│   │   ├── loyalty/          # Loyalty points & rewards
│   │   └── order-fulfillment/# Fulfillment orchestration
│   ├── scripts/              # One-off executable scripts
│   ├── shared/               # Shared utilities across the project
│   ├── subscribers/          # Event-driven subscribers
│   └── workflows/            # Custom workflow compositions
```

---

## Custom Modules

Each module under `src/modules/` follows a **ports-and-adapters (hexagonal)** architecture:

```
<module>/
├── adapters/    # Concrete implementations (DB repositories, external clients)
├── ports/       # Interfaces and service contracts
├── sdd/         # Module-level System Design Documents
└── __tests__/   # Unit tests
```

| Module | Responsibility |
|---|---|
| `loyalty` | Points accrual, redemption, expiry per customer/order |
| `admin-bff` | Aggregated admin data layer, custom admin queries |
| `commerce-catalog` | Product catalog extensions beyond core Medusa |
| `customer-identity` | Customer registration, authentication extensions |
| `checkout-payment` | Payment provider orchestration during checkout |
| `order-fulfillment` | Fulfillment workflow orchestration and provider routing |

---

## Getting Started

### Prerequisites

- Node.js ≥ 20
- PostgreSQL 14+
- Redis (required for background job queue and caching)

### Environment Variables

```env
DATABASE_URL=postgres://user:pass@localhost:5432/dtc_backend
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-jwt-secret
COOKIE_SECRET=your-cookie-secret
STORE_CORS=http://localhost:8000
ADMIN_CORS=http://localhost:9000
AUTH_CORS=http://localhost:9000,http://localhost:8000
PORT=9000
NODE_ENV=development
WORKER_MODE=shared
```

### Development

```bash
# Install dependencies
yarn install

# Run database migrations
yarn medusa db:migrate

# Seed initial data
yarn medusa exec ./src/migration-scripts/initial-data-seed.ts

# Start development server (with file watching)
yarn dev
# equivalent to: medusa develop
```

### Production Build

```bash
yarn build          # Compiles TypeScript → .medusa/server
yarn start          # medusa start
```

---

## API Surface

| Namespace | Base Path | Auth | Description |
|---|---|---|---|
| Store API | `/store/*` | Publishable key | Customer-facing endpoints |
| Admin API | `/admin/*` | Bearer JWT | Merchant dashboard endpoints |
| Custom Admin | `/admin/custom/*` | Bearer JWT | Project-specific admin routes |
| Custom Store | `/store/custom/*` | Publishable key | Project-specific store routes |
| Cloud Auth | `/cloud/auth/*` | — | Cloud deployment auth bridge |

---

## Testing

```bash
# Unit tests
yarn test:unit

# HTTP integration tests (requires running server + DB)
yarn test:integration:http

# Module integration tests
yarn test:integration:modules
```

---

## Container Deployment

The application is containerised via the project-level `Dockerfile`. The recommended production deployment separates HTTP and worker processes:

```yaml
# Simplified compose excerpt
services:
  backend-server:
    environment:
      WORKER_MODE: server
  backend-worker:
    environment:
      WORKER_MODE: worker
```

---

## Key Dependencies

| Package | Version | Purpose |
|---|---|---|
| `@medusajs/medusa` | 2.15.3 | Core framework |
| `@medusajs/framework` | 2.15.3 | DI container, HTTP, workflow engine |
| `@medusajs/dashboard` | 2.15.3 | Embedded admin UI |
| `@medusajs/admin-sdk` | 2.15.3 | Admin extension APIs |
| `@medusajs/cli` | 2.15.3 | `medusa` CLI |
| `typescript` | 5.6 | Language |
