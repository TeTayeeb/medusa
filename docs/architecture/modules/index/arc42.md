# arc42 Architecture Document — Index (Search) Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Index module must provide full-text and filtered product search for the storefront, keep the search index synchronised with the product catalogue via events, and support provider swap (built-in → Algolia/MeiliSearch) without changes to consuming code.

### 1.2 Quality Goals

| Priority | Quality Goal   | Scenario                                                         |
|----------|----------------|------------------------------------------------------------------|
| 1        | Accuracy       | Product index reflects the current catalogue within 5 seconds   |
| 2        | Replaceability | Swap to MeiliSearch via config with no storefront code changes   |
| 3        | Resilience     | Index provider downtime does not fail product create/update      |
| 4        | Performance    | Search responds in < 100ms for a 100,000-product catalogue       |

### 1.3 Stakeholders

| Role           | Expectation                                             |
|----------------|---------------------------------------------------------|
| Merchant       | Accurate, fast product search for customers             |
| Developer      | Simple provider swap; easy custom field indexing        |
| Devops         | No search infrastructure required for development       |

---

## 2. Architecture Constraints

- Index writes must be async and event-driven; never in the critical path of product mutations.
- The built-in provider must work without external services for local development.
- Provider failures must not propagate as errors to the product management API.

---

## 3. System Scope and Context

```
┌─────────────────┐     ┌───────────────────────────┐
│  Product Module │     │  Admin Dashboard           │
│  CRUD events    │     │  POST /admin/index/reindex │
└────────┬────────┘     └──────────────┬─────────────┘
         │ event bus                   │
┌────────▼─────────────────────────────▼─────────────┐
│                 Index Module                        │
│  IndexSubscriber + IndexModuleService               │
│  ProductDocumentTransformer                         │
│  ISearchProvider                                    │
└──────┬──────────────────────────────────────────────┘
       │
  ┌────▼────────────────┐  ┌──────────────────────┐
  │  Built-in Provider  │  │  Algolia / MeiliSearch│
  │  (PG tsvector)      │  │  (plugin provider)    │
  └─────────────────────┘  └──────────────────────┘
         ▲
┌────────┴─────────────────────────────────────────┐
│  GET /store/products?q=...                        │
│  (storefront search query)                        │
└──────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

- **Event-driven index writes**: `IndexSubscriber` listens for product events; index updates are fire-and-forget.
- **Provider interface**: All search backends implement `ISearchProvider`; consumers are decoupled from implementations.
- **Two-phase storefront query**: Search returns IDs; Product module returns hydrated data — ACL rules always applied.
- **Built-in PostgreSQL provider** for zero-infra development experience using tsvector full-text search.

---

## 5. Building Block View

### Level 1

```
IndexModule
  ├── IndexSubscriber              (event bus listener)
  ├── IndexModuleService           (reindex, provider management)
  ├── ProductDocumentTransformer   (DTO → flat search document)
  ├── ISearchProvider              (interface)
  ├── BuiltInSearchProvider        (PostgreSQL tsvector)
  └── [Plugin Providers]           (Algolia, MeiliSearch via plugins)
```

---

## 6. Runtime View

### Scenario: Product Updated → Index Synchronised

1. `PUT /admin/products/:id` completes successfully.
2. Product module emits `product.updated` on event bus.
3. `IndexSubscriber.handleProductUpsert()` called asynchronously.
4. Product fetched from Product module with full relations.
5. `ProductDocumentTransformer.transform()` produces flat document.
6. `ISearchProvider.replaceDocuments("products", [doc])` called.
7. Index updated; storefront search reflects new data within provider's latency SLA.

---

## 7. Deployment View

| Environment | Provider        | Infrastructure           |
|-------------|-----------------|--------------------------|
| Development | Built-in (PG)   | None; uses main DB       |
| Production  | MeiliSearch     | Dedicated MeiliSearch service |
| Production  | Algolia         | Algolia SaaS             |

---

## 8. Cross-Cutting Concerns

### Resilience
Index write failures are logged at `error` level but do not throw. A dead-letter retry queue is recommended for production (3 retries with exponential backoff).

### Visibility Filter
The two-phase query (search IDs → hydrate from Product module) ensures that draft/archived products returned by a stale index are filtered out by the Product module's status filter.

---

## 9. Architecture Decisions

| ID  | Decision                                    | Rationale                                                        |
|-----|---------------------------------------------|------------------------------------------------------------------|
| AD1 | Event-driven async index writes             | Keeps product mutation API latency unaffected by search backend  |
| AD2 | Two-phase storefront search                 | Search index may be slightly stale; Product module is authoritative |
| AD3 | Built-in PostgreSQL provider                | Zero infrastructure barrier for new developers                   |
| AD4 | Document transformer as separate component  | Allows custom providers to override document shape               |

---

## 10. Quality Scenarios

| Quality      | Scenario                                           | Measure                                       |
|--------------|----------------------------------------------------|-----------------------------------------------|
| Accuracy     | Product title updated                              | Search results updated within 5s              |
| Resilience   | MeiliSearch server unavailable                     | Product save succeeds; index retried later     |
| Performance  | Search query on 100k products                      | < 100ms (MeiliSearch); < 200ms (built-in)     |
| Replace      | Config changed to Algolia + restart                | Search works with Algolia; no code changes     |
