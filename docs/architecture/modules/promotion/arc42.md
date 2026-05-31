# arc42 Architecture Documentation — Promotion Module

**Module:** `@medusajs/promotion`  
**Version:** Medusa v2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The Promotion Module must:
- Allow merchants to configure percentage, fixed, and free-shipping discounts with flexible targeting (whole order, specific items, or shipping)
- Support buy-X-get-Y promotional mechanics
- Group promotions under campaigns with shared time windows and spend/usage budgets
- Evaluate complex eligibility rules against cart data without coupling to the cart domain
- Track campaign budget consumption atomically per order placement
- Support per-customer or per-entity budget limits (attribute-scoped budgets)

### 1.2 Quality Goals

| Quality Attribute | Priority | Scenario |
|---|---|---|
| Correctness | Critical | Discount amounts must be precisely computed; over-discounting must be impossible |
| Performance | High | `computeActions` must complete in <50ms for typical carts (5–20 items, 1–5 promotions) |
| Extensibility | Medium | New rule attributes must require no core module changes |
| Isolation | High | Promotion logic must not depend on cart or order module internals |

---

## 2. Architecture Constraints

- The module must be usable as a standalone library in non-Medusa contexts
- PostgreSQL is the only supported persistence backend
- The module must not issue HTTP requests; all external dependencies are injected via the container
- Soft-delete is required for all entities to preserve audit history

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────┐
│                   Medusa Application                │
│                                                     │
│  ┌──────────┐    computeActions()    ┌───────────┐ │
│  │   Cart   │ ──────────────────────► Promotion  │ │
│  │  Module  │ ◄────────────────────── Module     │ │
│  └──────────┘    DiscountActions     └───────────┘ │
│                                           │         │
│  ┌──────────┐    registerUsage()          │         │
│  │  Order   │ ──────────────────────────►│         │
│  │  Module  │                            │         │
│  └──────────┘                      ┌─────▼──────┐  │
│                                    │ PostgreSQL  │  │
│                                    └────────────┘  │
└─────────────────────────────────────────────────────┘
```

### External Interfaces

| Interface | Direction | Description |
|---|---|---|
| `computeActions(codes, cartData)` | Inbound | Cart module provides cart snapshot for evaluation |
| `registerUsage(actions)` | Inbound | Order module registers discount actions after placement |
| `IPromotionModuleService` | Outbound | Admin API workflows use CRUD operations |
| PostgreSQL | Outbound | Persistence of all entities |

---

## 4. Solution Strategy

The module adopts a **pure computation model** for discount evaluation: `computeActions` is stateless with respect to the cart — it reads promotion configuration, evaluates rules, and returns discount actions without writing any cart state. This enables safe concurrent cart recalculations.

Budget tracking uses an **optimistic counter pattern**: `registerUsage` increments budget counters after a successful order commit. This avoids distributed locks but may permit slight over-redemption under extreme concurrency. Merchants can compensate by setting conservative limits.

---

## 5. Building Blocks

### Level 1: Module Structure

```
@medusajs/promotion
├── services/
│   └── promotion-module.ts      # IPromotionModuleService implementation
├── models/
│   ├── promotion.ts
│   ├── campaign.ts
│   ├── campaign-budget.ts
│   ├── campaign-budget-usage.ts
│   ├── application-method.ts
│   ├── promotion-rule.ts
│   └── promotion-rule-value.ts
└── migrations/
```

### Level 2: Key Responsibilities

| Component | Responsibility |
|---|---|
| `PromotionModuleService` | Orchestrates all CRUD, rule evaluation, and usage tracking |
| `ApplicationMethodEvaluator` | Computes discount amounts for each target type and allocation strategy |
| `RuleEngine` | Evaluates `PromotionRule` predicates against cart context data |
| `BudgetTracker` | Manages campaign budget consumption and per-attribute limit enforcement |

---

## 6. Runtime View

### Scenario: Coupon Applied to Cart

```
1. Customer enters code "SUMMER20"
2. Cart workflow calls computeActions(["SUMMER20"], cartSnapshot)
3. Module resolves Promotion record for code
4. Module checks campaign date range → valid
5. Module evaluates promotion-level rules:
   - customer_group_id IN [vip_group] → PASS (customer is VIP)
6. Module evaluates ApplicationMethod:
   - type: percentage, value: 20, target_type: items
   - target_rules: product_category_id IN [apparel]
   - Filters cart items to apparel items only
   - Computes 20% of filtered items total = $12.00
7. Returns: [{ action: "addItemAdjustment", item_id: "...", amount: 1200 }]
8. Cart module applies adjustments to line items
```

---

## 7. Deployment View

The Promotion Module runs in the same Node.js process as the Medusa application. It does not have network boundaries from the application layer. Database access is via MikroORM entity manager with connection pooling.

---

## 8. Crosscutting Concepts

### Soft-Delete
All entities use a `deleted_at` nullable timestamp column. Unique constraints on codes/identifiers use `WHERE deleted_at IS NULL` partial indexes.

### Metadata
All primary entities expose a `metadata: json | null` field for arbitrary extension data, avoiding schema changes for custom merchant attributes.

### Event Emission
The module emits domain events via `@EmitEvents()` on mutating service methods. Subscribers can hook into `promotion.created`, `promotion.deleted`, `campaign.created`, etc.

### Searchable Fields
`Promotion.code`, `Campaign.name`, `Campaign.description` are marked `searchable()`, enabling them to be indexed by the Medusa search module (Meilisearch, Algolia, etc.).

---

## 9. Architecture Decisions

### ADR-1: Rule Engine as Data (not Code)
Rules are stored as data (`attribute + operator + values`) rather than code callbacks. This enables admin UI rule builders, serialization, and avoids arbitrary code execution security risks.

### ADR-2: computeActions is Stateless
Discount computation does not write any database state. This design enables safe retries, parallel execution, and separation of read (compute) from write (registerUsage) concerns.

### ADR-3: Campaign Budget Counter (Not Reservation)
Budget usage is tracked via post-purchase increment rather than a reservation/pre-authorization model. This simplifies the happy path at the cost of potential slight over-redemption at very high concurrency.

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Unique promotion codes | Partial unique index enforces this at DB level |
| Audit trail | All entities carry `created_at`, `updated_at`, soft-delete `deleted_at` |
| Searchability | Searchable fields are indexed for full-text search |
| Rule extensibility | New attributes require no schema changes — add a new string value to `attribute` |
