# @dtc/backend — System Design Document

> Version: 1.0 | Medusa: 2.15.3 | Status: Current

---

## 1. Purpose and Scope

This document describes the internal design of the `@dtc/backend` Medusa application — the deployable server that powers the DTC commerce platform. It covers startup sequencing, request lifecycle, worker process separation, module extension patterns, configuration reference, and all defined extension points.

---

## 2. Startup Sequence

When `medusa start` (or `medusa develop`) is invoked, the framework bootstraps in the following order:

```
1. Load environment variables   (dotenv via loadEnv())
2. Load medusa-config.ts        (defineConfig — projectConfig, modules)
3. Initialise IoC container     (Medusa DI container, Awilix)
4. Register core modules        (Product, Order, Cart, Customer, … built-in)
5. Register custom modules      (loyalty, admin-bff, commerce-catalog, …)
6. Run module migrations        (if MEDUSA_FF_MEDUSA_V2 or migrate flag set)
7. Register module links        (src/links/* — cross-module FK definitions)
8. Load subscribers             (src/subscribers/* — event handlers)
9. Load scheduled jobs          (src/jobs/* — cron-style background tasks)
10. Build HTTP router            (Medusa auto-discovers src/api/** routes)
11. Bind HTTP server            (Express on PORT, default 9000)
12. Emit `application.started`  (subscribers may react)
```

In **worker mode** (`WORKER_MODE=worker`), steps 10–11 are skipped. In **server mode** (`WORKER_MODE=server`), steps 8–9 are skipped.

---

## 3. Configuration Reference (`medusa-config.ts`)

```typescript
import { loadEnv, defineConfig } from '@medusajs/framework/utils'

loadEnv(process.env.NODE_ENV || 'development', process.cwd())

module.exports = defineConfig({
  projectConfig: {
    databaseUrl: process.env.DATABASE_URL,   // Required: PostgreSQL connection
    http: {
      storeCors:  process.env.STORE_CORS!,   // Storefront origin(s)
      adminCors:  process.env.ADMIN_CORS!,   // Admin dashboard origin(s)
      authCors:   process.env.AUTH_CORS!,    // Auth endpoint allowed origins
      jwtSecret:  process.env.JWT_SECRET  || 'supersecret',
      cookieSecret: process.env.COOKIE_SECRET || 'supersecret',
    },
  },
  // modules: [] — custom modules registered here
})
```

### Environment Variable Reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | ✅ | — | PostgreSQL DSN |
| `REDIS_URL` | ✅ (worker/jobs) | — | Redis DSN for queue and cache |
| `JWT_SECRET` | ✅ (prod) | `supersecret` | JWT signing secret |
| `COOKIE_SECRET` | ✅ (prod) | `supersecret` | Cookie signing secret |
| `STORE_CORS` | ✅ | — | Allowed storefront origins |
| `ADMIN_CORS` | ✅ | — | Allowed admin origins |
| `AUTH_CORS` | ✅ | — | Allowed auth origins |
| `PORT` | ❌ | `9000` | HTTP listen port |
| `NODE_ENV` | ❌ | `development` | Runtime environment |
| `WORKER_MODE` | ❌ | `shared` | `shared` / `server` / `worker` |

---

## 4. Request Lifecycle

```
HTTP Request
    │
    ▼
Express Middleware Stack
    ├── CORS (per-namespace: store / admin / auth)
    ├── Body parser (JSON)
    ├── Authentication middleware
    │       ├── /store/*  → validate publishable key → attach store context
    │       ├── /admin/*  → validate JWT Bearer     → attach admin actor
    │       └── /auth/*   → public (no auth required)
    │
    ▼
Route Handler  (src/api/**/<method>.ts  or  @medusajs/medusa built-in)
    │
    ├── Resolve services via req.scope  (Awilix container)
    ├── Execute workflow  (createWorkflow → run())
    │       ├── Steps resolve module services from container
    │       ├── Steps are transactional (each step has compensate())
    │       └── Workflow commits or compensates on failure
    │
    ▼
Response  (res.json())
```

### Custom Route Example

```typescript
// src/api/store/custom/route.ts
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"

export async function GET(req: MedusaRequest, res: MedusaResponse) {
  const loyaltyService = req.scope.resolve("loyalty")
  const balance = await loyaltyService.getBalance(req.auth_context.customer_id)
  res.json({ balance })
}
```

---

## 5. Custom Module Architecture

All six custom modules share a **hexagonal (ports-and-adapters)** internal structure:

```
src/modules/<module>/
├── ports/
│   ├── IService.ts          # Primary port (service interface)
│   └── IRepository.ts       # Secondary port (persistence interface)
├── adapters/
│   ├── medusa/              # MedusaService-based implementation
│   └── persistence/         # MikroORM entity + repository
└── __tests__/               # Unit + integration tests
```

#### Module Registration Pattern

```typescript
// medusa-config.ts
import { Modules } from "@medusajs/framework/utils"

defineConfig({
  modules: [
    {
      resolve: "./src/modules/loyalty",
      options: { pointsPerCurrency: 1 },
    },
  ],
})
```

#### Module Link Pattern

```typescript
// src/links/loyalty-customer.ts
import { defineLink } from "@medusajs/framework/utils"
import LoyaltyModule from "../modules/loyalty"
import { Modules } from "@medusajs/framework/utils"

export default defineLink(
  LoyaltyModule.linkable.loyaltyAccount,
  { linkable: Modules.CUSTOMER + ".customer", isList: false }
)
```

---

## 6. Workflow and Step Pattern

Custom workflows in `src/workflows/` compose steps to implement multi-module business logic with built-in compensation (saga pattern):

```typescript
// src/workflows/award-loyalty-points.ts
import { createWorkflow, createStep, StepResponse, WorkflowResponse } from "@medusajs/framework/workflows-sdk"

const calculatePointsStep = createStep(
  "calculate-points",
  async ({ orderId, total }, { container }) => {
    const loyalty = container.resolve("loyalty")
    const points = await loyalty.calculatePoints(total)
    return new StepResponse({ points }, { orderId })
  }
)

export const awardLoyaltyPointsWorkflow = createWorkflow(
  "award-loyalty-points",
  (input: WorkflowData<{ orderId: string; customerId: string; total: number }>) => {
    const { points } = calculatePointsStep(input)
    return new WorkflowResponse({ awarded: points })
  }
)
```

---

## 7. Subscriber Pattern

```typescript
// src/subscribers/order-placed.ts
import { SubscriberArgs, SubscriberConfig } from "@medusajs/framework"
import { awardLoyaltyPointsWorkflow } from "../workflows/award-loyalty-points"

export default async function orderPlacedHandler({ event, container }: SubscriberArgs) {
  const { id: orderId, customer_id, total } = event.data
  await awardLoyaltyPointsWorkflow(container).run({
    input: { orderId, customerId: customer_id, total },
  })
}

export const config: SubscriberConfig = {
  event: "order.placed",
}
```

---

## 8. Worker Process Separation

For production deployments, the recommended topology separates concerns:

| Process | `WORKER_MODE` | Handles |
|---|---|---|
| `backend-server` | `server` | HTTP requests, API responses |
| `backend-worker` | `worker` | Subscribers, scheduled jobs, async workflows |

Both processes share the same Docker image and codebase — only `WORKER_MODE` differs. The Redis queue decouples the two processes. This enables independent scaling of the HTTP tier and the background processing tier.

---

## 9. Admin Extensions

Admin dashboard extensions live in `src/admin/`:

- **`widgets/`** — React components injected into existing admin pages via the `@medusajs/admin-sdk` injection zones API.
- **`i18n/`** — Internationalisation string overrides for the admin UI.

```typescript
// src/admin/widgets/loyalty-widget.tsx
import { defineWidgetConfig } from "@medusajs/admin-sdk"
export const config = defineWidgetConfig({ zone: "customer.details.after" })
export default function LoyaltyWidget({ data }) {
  return <div>Points: {data.loyalty_balance}</div>
}
```

---

## 10. Data Seeding

`src/migration-scripts/initial-data-seed.ts` is an executable Medusa script that provisions baseline commerce data on first run:

- Store, sales channels, API keys
- Regions with tax rules (GB, DE, DK, SE, FR, ES, IT)
- Shipping profiles and options
- Stock locations and inventory levels
- Product categories and collections
- Sample products
