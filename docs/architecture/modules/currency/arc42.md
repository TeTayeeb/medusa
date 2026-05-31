# Architecture Documentation — Currency Module (arc42)

## 1. Introduction and Goals

The Currency module (`@medusajs/currency`) addresses the need for a single authoritative source of ISO 4217 currency definitions within the Medusa platform. Without it, each module handling monetary data would independently define or duplicate currency metadata, creating the risk of inconsistent decimal precision or symbol representation.

**Quality Goals:**

| Priority | Quality Goal  | Description                                                       |
|----------|---------------|-------------------------------------------------------------------|
| 1        | Consistency   | All modules reference identical currency definitions              |
| 2        | Simplicity    | No write API reduces attack surface and cognitive complexity      |
| 3        | Performance   | Fast reads; the dataset is small and read-heavy                   |
| 4        | Replaceability | Module interface allows substitution with a custom implementation |

## 2. Constraints

- Currency codes must conform to the ISO 4217 standard (three uppercase letters).
- Data is seeded from an authoritative source, not user-defined, to prevent corruption.
- The module must implement the `ICurrencyModuleService` interface for platform compatibility.
- No external API calls are made; data is local and immutable.

## 3. Context and Scope

```
┌─────────────────────────────────────────────────────────────────┐
│                        Medusa Platform                          │
│                                                                 │
│  [Admin API] ──GET /admin/currencies──► [Currency Module]       │
│  [Region Module] ──validates code──────► [Currency Module]      │
│  [Pricing Module] ──references code────► [Currency Module]      │
│  [Store Module] ──default currency─────► [Currency Module]      │
│                                                                 │
│  [Currency Module] ──reads──► [PostgreSQL: currency table]      │
└─────────────────────────────────────────────────────────────────┘
```

The Currency module has **no outbound dependencies** on other business modules. It is a pure data provider sitting at the base of the monetary data hierarchy.

## 4. Solution Strategy

| Challenge                              | Strategy                                                                    |
|----------------------------------------|-----------------------------------------------------------------------------|
| Ensuring consistent decimal precision  | Single source of truth; all modules read `decimal_digits` from this module  |
| Avoiding duplication                   | No embedded currency data in other modules; only `currency_code` string stored |
| Preventing accidental mutation         | No POST/PUT/DELETE API routes defined; data changed only via seed scripts    |
| Supporting custom currency sets        | Module can be replaced with a custom implementation via Medusa module config |

## 5. Building Block View

```
Currency Module
├── HTTP Layer
│   └── Admin Route Handler (GET /admin/currencies, GET /admin/currencies/:code)
│
├── Service Layer
│   └── CurrencyModuleService
│       ├── listCurrencies()
│       ├── listAndCountCurrencies()
│       └── retrieveCurrency()
│
├── Data Access Layer
│   └── CurrencyRepository (MikroORM EntityRepository<Currency>)
│
├── Domain Model
│   └── Currency (MikroORM Entity)
│
└── Seed Script
    └── Upserts ~170 ISO 4217 currencies on boot
```

## 6. Runtime View

**Scenario A: Admin fetches currency list**

```
Browser → GET /admin/currencies
  → Authenticate (JWT middleware)
  → Route Handler resolves ICurrencyModuleService from container
  → CurrencyModuleService.listCurrencies(filters, config)
  → MikroORM: SELECT code, name, symbol... FROM currency ORDER BY code
  → Serialize to CurrencyDTO[]
  → JSON response { currencies: [...], count, limit, offset }
```

**Scenario B: Region module validates a currency code**

```
RegionModuleService.createRegions({ currency_code: "EUR" })
  → useQueryGraphStep OR direct container.resolve(Modules.CURRENCY)
  → CurrencyModuleService.retrieveCurrency("EUR")
  → Found → proceed with region creation
  → Not found → throw MedusaError.Types.NOT_FOUND → 400 response
```

## 7. Deployment View

The Currency module is deployed as an integral part of the Medusa server process. It shares:
- The main PostgreSQL database (single `currency` table).
- The application's IoC container (resolved via `Modules.CURRENCY`).
- The HTTP server (routes registered under `/admin/currencies`).

No separate deployment artifact, sidecar, or microservice is required.

## 8. Cross-Cutting Concerns

| Concern        | Approach                                                          |
|----------------|-------------------------------------------------------------------|
| Authentication | Admin routes protected by JWT authentication middleware           |
| Transactions   | Read-only; no transaction management needed                       |
| Logging        | Standard Medusa logger; errors at WARN level                     |
| Observability  | Standard HTTP request logging via Medusa middleware               |
| Error handling | `MedusaError.Types.NOT_FOUND` for missing codes; propagates as 404 |

## 9. Design Decisions

| ID  | Decision                           | Rationale                                                                       | Alternatives Considered          |
|-----|------------------------------------|---------------------------------------------------------------------------------|----------------------------------|
| D1  | Natural PK (`code` string)         | ISO code is globally unique and already used as the reference in other modules  | UUID PK with code unique index   |
| D2  | Read-only API (no POST/DELETE)      | Reference data should not be modified via API to prevent data corruption        | Allow admin writes with warnings |
| D3  | Idempotent seed at startup          | Safe to run on every deploy without manual migration steps                      | One-time migration only          |
| D4  | Soft delete support                | Allows graceful deprecation without breaking references in historical records   | Hard delete only                 |

## 10. Risks and Technical Debt

| Risk                               | Mitigation                                                  |
|------------------------------------|-------------------------------------------------------------|
| ISO 4217 standard adds new currency | Re-run seed script; idempotent upsert handles new entries  |
| Custom currency needs (crypto, etc.)| Module can be replaced; or seed extended with custom codes |
| Cache staleness if data changes     | TTL-based cache invalidation; currency data changes rarely  |
