# Architecture Documentation — Region Module (arc42)

## 1. Introduction and Goals

The Region module is the primary market configuration unit in Medusa. It allows a single Medusa instance to serve multiple geographically or commercially distinct markets, each with its own currency, tax defaults, and provider availability. It is consumed by the Cart, Order, Tax, and Pricing modules.

**Quality Goals:**

| Priority | Quality Goal    | Description                                                              |
|----------|-----------------|--------------------------------------------------------------------------|
| 1        | Market Isolation | Each region's configuration changes do not bleed into other regions     |
| 2        | Flexibility     | Support arbitrary country groupings, not just standard continents        |
| 3        | Consistency     | Cart and Order snapshots preserve region data at time of purchase        |
| 4        | Availability    | Store API must be readable without authentication for storefronts        |

## 2. Constraints

- A country can belong to at most one region at a time (enforced by unique index).
- `currency_code` must reference a valid currency from the Currency module.
- Regions cannot be fully deleted if they are referenced by active orders (soft delete only).

## 3. Context and Scope

```
External Systems:
  [Admin Browser] ──CRUD──► [Admin API /admin/regions] ──► [Region Module]
  [Storefront]    ──READ──► [Store API /store/regions]  ──► [Region Module]

Internal Dependencies:
  [Region Module] ──validates currency_code──► [Currency Module]
  [Cart Module]   ──reads region config──────► [Region Module]
  [Tax Module]    ──linked TaxRegions────────► [Region Module]
  [Payment Module]──scoped by region─────────► via module links
```

## 4. Solution Strategy

| Challenge                              | Strategy                                               |
|----------------------------------------|--------------------------------------------------------|
| Multi-currency storefronts             | Currency code stored per region; Cart resolves it      |
| Tax complexity per market              | Region stores defaults; detailed rules in Tax module   |
| Provider availability per market       | Module links (not Region internals) manage associations |
| Historical accuracy of completed orders| Cart/Order snapshot region data at checkout completion  |

## 5. Building Block View

```
Region Module
├── HTTP Layer
│   ├── Admin Routes (GET/POST/DELETE /admin/regions)
│   └── Store Routes (GET /store/regions)
│
├── Workflow Layer
│   ├── createRegionsWorkflow
│   ├── updateRegionsWorkflow
│   └── deleteRegionsWorkflow
│
├── Service Layer
│   └── RegionModuleService
│       ├── createRegions / updateRegions / deleteRegions
│       ├── listRegions / retrieveRegion
│       └── upsertRegionCountries / deleteRegionCountries
│
├── Data Access Layer
│   ├── RegionRepository
│   └── RegionCountryRepository
│
└── Domain Model
    ├── Region
    └── RegionCountry
```

## 6. Runtime View

**Scenario A: Admin creates a region**

```
POST /admin/regions { name, currency_code, countries: ["DE","AT"] }
  → Auth middleware (admin JWT)
  → Route handler calls createRegionsWorkflow(req.scope).run(input)
  → createRegionsStep: validates currency_code via remote query
  → RegionModuleService.createRegions() (within transaction)
  → upsertRegionCountries() for each country code
  → Hook emitted: regionsCreated
  → Response: { region: RegionDTO }
```

**Scenario B: Cart creation resolves region**

```
POST /store/carts { region_id: "reg_01HXXX" }
  → CartModuleService resolves Region via remote query
  → Reads currency_code → all prices rendered in that currency
  → Reads automatic_taxes → whether to call tax module at checkout
  → Cart created with region_id stored
```

## 7. Deployment View

Deployed as part of the Medusa server process. Shares the PostgreSQL database (`region` and `region_country` tables). No separate process required.

## 8. Cross-Cutting Concerns

| Concern        | Approach                                                                    |
|----------------|-----------------------------------------------------------------------------|
| Authentication | Admin routes: JWT required. Store routes: public (read-only)                |
| Transactions   | `@InjectTransactionManager()` on write operations; atomic region + country writes |
| Events         | Domain hooks (`regionsCreated`, etc.) emitted from workflows                |
| Soft delete    | `deleted_at` on Region; country records cascade on region delete            |

## 9. Design Decisions

| ID  | Decision                              | Rationale                                                                |
|-----|---------------------------------------|--------------------------------------------------------------------------|
| D1  | Store API is read-only                 | Customers select regions; merchants configure them in the Admin          |
| D2  | Countries in separate table           | Avoids JSON arrays; enables indexed lookups by `iso_2`                   |
| D3  | One country per region (unique index)  | Prevents ambiguous region resolution for a customer's country            |
| D4  | Tax defaults on Region                 | Convenience field; detailed tax rules live in the Tax module             |

## 10. Risks and Technical Debt

| Risk                                    | Mitigation                                                  |
|-----------------------------------------|-------------------------------------------------------------|
| Currency module unavailable at region create | Validation step in workflow throws and rolls back     |
| Region delete breaking Cart references  | Soft delete; orders/carts retain region_id snapshot         |
| Province-level tax not covered by Region| Province-level rules handled in Tax module TaxRegion hierarchy |
