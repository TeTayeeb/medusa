# arc42 Architecture Document — Link Modules

## 1. Introduction and Goals

### 1.1 Requirements Overview
Link Modules must enable cross-module entity relationships without introducing direct database foreign keys between module schemas, support bidirectional query traversal, integrate with the framework's Query helper, and handle cascading deletions safely.

### 1.2 Quality Goals

| Priority | Quality Goal   | Scenario                                                              |
|----------|----------------|-----------------------------------------------------------------------|
| 1        | Isolation      | Removing the Sales Channel module does not require schema changes to the Product module |
| 2        | Correctness    | Deleting a product soft-deletes all its sales channel link records    |
| 3        | Discoverability| A developer can query `product.sales_channels` as a natural graph path |
| 4        | Performance    | Cross-module query with one link traversal resolves in < 10ms         |

### 1.3 Stakeholders

| Role           | Expectation                                                  |
|----------------|--------------------------------------------------------------|
| Module author  | Define a link in a few lines; framework handles the rest     |
| Developer      | Query linked data via `query.graph()` as if it were native   |
| Platform admin | Cross-module relationships are durable and cascade correctly |

---

## 2. Architecture Constraints

- No SQL foreign keys may reference tables owned by a different module.
- Link tables are owned by neither of the linked modules; they are independent module-level artefacts.
- The Query helper is the only supported mechanism for cross-module data retrieval.

---

## 3. System Scope and Context

```
┌─────────────┐         ┌──────────────────────────────┐         ┌──────────────────┐
│   Module A  │         │        Link Module           │         │    Module B      │
│   Product   │◄───────►│  product_sales_channel table │◄───────►│  SalesChannel   │
│  (product_id)│        │  (product_id, sc_id)         │         │  (sales_ch_id)   │
└─────────────┘         └──────────────────────────────┘         └──────────────────┘
                                       ▲
                         ┌─────────────┴───────────────┐
                         │   Query Helper               │
                         │   query.graph({ entity:      │
                         │     "product", fields:       │
                         │     ["sales_channels.*"] })  │
                         └─────────────────────────────┘
```

---

## 4. Solution Strategy

- **`defineLink` API**: Declarative link definition generates pivot table DDL, IoC bindings, and query graph nodes automatically.
- **RemoteLinkService**: Centralised CRUD for link records; decouples consuming code from pivot table names.
- **Cascade via events**: Parent module deletion events trigger link soft-deletes through event subscribers.
- **Lexicographic key ordering** in multi-key operations prevents deadlocks.

---

## 5. Building Block View

### Level 1

```
LinkModules (framework)
  ├── defineLink()                  (link definition factory)
  ├── RemoteLinkService             (create / delete / dismiss links)
  ├── ModuleJoinerConfig            (query graph registration)
  ├── LinkCascadeSubscriber         (handles deletion event cascade)
  └── [Generated pivot tables]      (one per defined link)
```

### Level 2 — defineLink output

```
defineLink(Product.linkable.product, SalesChannel.linkable.salesChannel)
  → ModuleJoinerConfig {
      tableName: "product_sales_channel",
      schema: { product_id, sales_channel_id, data, timestamps },
      joiner: { entity: "product", related: "sales_channel", through: table }
    }
```

---

## 6. Runtime View

### Scenario: Querying Product with Sales Channels

1. `query.graph({ entity: "product", fields: ["id", "title", "sales_channels.*"] })`.
2. Query helper resolves `sales_channels` field → looks up `ProductSalesChannelLink` joiner config.
3. SQL: `SELECT sales_channel_id FROM product_sales_channel WHERE product_id IN (...)`.
4. SQL: `SELECT * FROM sales_channel WHERE id IN (...)` (via SalesChannel module service).
5. Results merged into product objects.
6. Single response returned to caller.

### Scenario: Product Deleted → Links Cascade

1. `productService.softDelete(["prod_01"])`.
2. `product.deleted` event emitted.
3. `LinkCascadeSubscriber` receives event.
4. `remoteLink.softDelete([{ product_id: "prod_01" }])` called for all linked pivot tables.
5. All `product_sales_channel`, `product_category` etc. records soft-deleted.

---

## 7. Deployment View

Link tables are in the same PostgreSQL database as their parent module tables. The Link Module framework code runs in-process with the Medusa server. No external services required.

---

## 8. Cross-Cutting Concerns

### Orphan Cleanup
Hard deletes on a parent entity may leave orphaned pivot records if the cascade event fails. A scheduled maintenance job periodically cleans orphaned records via a `JOIN ... WHERE parent IS NULL` query.

### Index Strategy
Every pivot table has indices on both FK columns. The unique constraint on `(entity_a_id, entity_b_id)` prevents duplicate links.

---

## 9. Architecture Decisions

| ID  | Decision                                    | Rationale                                                        |
|-----|---------------------------------------------|------------------------------------------------------------------|
| AD1 | Separate pivot tables per link              | Avoids EAV anti-pattern; preserves type safety and index efficiency |
| AD2 | Links owned by neither module               | Both modules remain deployable independently                     |
| AD3 | Cascade via event bus, not DB triggers      | Keeps cascade logic in application layer; easier to test and extend |
| AD4 | Query helper as sole cross-module query API | Centralises link resolution; prevents ad-hoc raw SQL across modules |

---

## 10. Quality Scenarios

| Quality    | Scenario                                              | Measure                                       |
|------------|-------------------------------------------------------|-----------------------------------------------|
| Isolation  | SalesChannel module removed from deployment           | Product module starts without errors          |
| Correctness| Product deleted → check pivot table                  | `deleted_at` set on all product_sales_channel rows |
| Performance| Query 50 products with sales_channels                 | < 10ms (2 indexed queries)                    |
