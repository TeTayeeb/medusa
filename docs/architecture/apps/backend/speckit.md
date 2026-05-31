# @dtc/backend — SpecKit

> Specification inventory for the Medusa backend application
> Version: 1.0 | Medusa: 2.15.3

---

## 1. Overview

This document provides a concise, structured specification inventory for `@dtc/backend`. It defines functional requirements, API contracts, data models, non-functional requirements, and acceptance criteria for the custom extensions built on top of Medusa v2.

---

## 2. Functional Specifications

### 2.1 Store API

#### `GET /store/custom`
- **Purpose**: Placeholder for project-specific storefront data endpoint
- **Auth**: Publishable API key (`x-publishable-api-key` header)
- **Response**: `200 OK`
- **Extension point**: Implement business logic in `src/api/store/custom/route.ts`

#### `GET /admin/custom`
- **Purpose**: Placeholder for project-specific admin data endpoint
- **Auth**: Bearer JWT (admin session)
- **Response**: `200 OK`
- **Extension point**: Implement business logic in `src/api/admin/custom/route.ts`

#### `GET /cloud/auth/*`
- **Purpose**: Cloud deployment auth bridge endpoints
- **Auth**: Context-dependent
- **Location**: `src/api/cloud/auth/`

---

### 2.2 Loyalty Module Specification

#### Functional Requirements

| ID | Requirement |
|---|---|
| LY-01 | System SHALL track loyalty points per customer account |
| LY-02 | System SHALL award points on order completion based on configurable `pointsPerCurrency` ratio |
| LY-03 | System SHALL allow customers to redeem points during checkout as a discount |
| LY-04 | System SHALL expire unused points after a configurable period |
| LY-05 | System SHALL provide a loyalty balance and transaction history per customer |
| LY-06 | System SHALL link loyalty accounts to Medusa customer records via module link |

#### Data Model

```typescript
interface LoyaltyAccount {
  id: string
  customer_id: string       // FK → Customer module
  balance: number           // Current unredeemed points
  lifetime_earned: number   // Total points ever earned
  created_at: Date
  updated_at: Date
}

interface LoyaltyTransaction {
  id: string
  account_id: string
  order_id?: string         // Source order (nullable for manual adjustments)
  type: "earn" | "redeem" | "expire" | "adjust"
  points: number            // Positive = earn; negative = redeem/expire
  created_at: Date
}
```

#### Service Contract (`ILoyaltyService`)

```typescript
interface ILoyaltyService {
  getBalance(customerId: string): Promise<number>
  getTransactionHistory(customerId: string, pagination: Pagination): Promise<LoyaltyTransaction[]>
  awardPoints(customerId: string, orderId: string, orderTotal: number): Promise<LoyaltyTransaction>
  redeemPoints(customerId: string, points: number): Promise<LoyaltyTransaction>
  expirePoints(beforeDate: Date): Promise<number>  // Returns count of expired records
}
```

---

### 2.3 Order Fulfilment Module Specification

| ID | Requirement |
|---|---|
| OF-01 | System SHALL route fulfilment requests to the appropriate provider based on item attributes |
| OF-02 | System SHALL update order fulfilment status on provider webhooks |
| OF-03 | System SHALL support multiple concurrent fulfilment providers |
| OF-04 | System SHALL expose fulfilment status changes as Medusa events |

---

### 2.4 Customer Identity Module Specification

| ID | Requirement |
|---|---|
| CI-01 | System SHALL support email/password authentication via Medusa's auth module |
| CI-02 | System SHALL optionally support OAuth provider pass-through |
| CI-03 | System SHALL enforce password complexity policy (min 8 chars, 1 number, 1 uppercase) |
| CI-04 | System SHALL emit `customer.registered` event on new account creation |

---

## 3. Non-Functional Requirements

| ID | Category | Requirement | Target |
|---|---|---|---|
| NFR-01 | Performance | P95 API response time | < 300 ms |
| NFR-02 | Performance | P50 API response time | < 100 ms |
| NFR-03 | Availability | Monthly uptime | ≥ 99.9 % |
| NFR-04 | Security | Auth token expiry | 24 h (JWT), 7 d (cookie) |
| NFR-05 | Security | All secrets in environment | No hardcoded secrets |
| NFR-06 | Scalability | Horizontal scaling | HTTP tier stateless |
| NFR-07 | Data | Backup frequency | Daily automated PostgreSQL dump |
| NFR-08 | Compliance | PII data residency | EU region PostgreSQL instance |

---

## 4. Configuration Specification

### Required Environment Variables

```env
DATABASE_URL        # Format: postgres://user:pass@host:5432/db  — REQUIRED
REDIS_URL           # Format: redis://host:6379                  — REQUIRED (workers)
JWT_SECRET          # Min 32 chars                               — REQUIRED (prod)
COOKIE_SECRET       # Min 32 chars                               — REQUIRED (prod)
STORE_CORS          # Comma-separated origins                    — REQUIRED
ADMIN_CORS          # Comma-separated origins                    — REQUIRED
AUTH_CORS           # Comma-separated origins                    — REQUIRED
```

### Optional Environment Variables

```env
PORT=9000           # Default: 9000
NODE_ENV=production # Default: development
WORKER_MODE=shared  # Options: shared | server | worker
```

---

## 5. Worker Mode Acceptance Criteria

### WORKER_MODE=server

- [ ] HTTP server binds to `PORT` and responds to requests
- [ ] No subscriber or scheduled job processing occurs
- [ ] `GET /health` returns `200 OK`

### WORKER_MODE=worker

- [ ] No HTTP server is started
- [ ] Subscribers consume events from Redis queue
- [ ] Scheduled jobs execute at configured intervals
- [ ] Worker exits cleanly on `SIGTERM`

### WORKER_MODE=shared (default)

- [ ] Both HTTP and worker behaviours are present
- [ ] Graceful shutdown: in-flight requests complete before process exits

---

## 6. Build and Deployment Specification

### Build Output

```
.medusa/
└── server/          # Compiled output from medusa build
    ├── index.js
    └── ...
```

### Build Command Matrix

| Command | Purpose | Output |
|---|---|---|
| `yarn build` | Production compilation | `.medusa/server/` |
| `yarn dev` | Development with watch | In-memory (nodemon) |
| `yarn start` | Start compiled output | Process on `PORT` |
| `yarn test:unit` | Unit test suite | Jest results |
| `yarn test:integration:http` | HTTP integration tests | Jest results |
| `yarn test:integration:modules` | Module integration tests | Jest results |

---

## 7. Extension Points

| Extension Point | Location | Description |
|---|---|---|
| Custom API route | `src/api/{admin,store,cloud}/<path>/route.ts` | Export named HTTP method functions |
| Custom module | `src/modules/<name>/` + `medusa-config.ts` | Implement ports/adapters; register in config |
| Module link | `src/links/<name>.ts` | Define cross-module FK with `defineLink()` |
| Workflow | `src/workflows/<name>.ts` | Compose `createWorkflow()` with steps |
| Subscriber | `src/subscribers/<name>.ts` | Export handler + `config.event` |
| Scheduled job | `src/jobs/<name>.ts` | Export handler + `config.schedule` |
| Admin widget | `src/admin/widgets/<name>.tsx` | Export component + `defineWidgetConfig()` |
| Migration script | `src/migration-scripts/<name>.ts` | Export async `default` function |
