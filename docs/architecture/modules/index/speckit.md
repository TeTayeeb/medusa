# SpecKit — Index (Search) Module

## Module Identity

| Attribute      | Value                                      |
|----------------|--------------------------------------------|
| Module Name    | `@medusajs/index`                          |
| Version        | 2.15.4                                     |
| Module Key     | `index`                                    |
| Type           | Infrastructure / Search                    |
| Database Tables| `search_document` (built-in provider only) |
| Event Emitter  | No                                         |
| Event Consumer | Yes (`product.created`, `product.updated`, `product.deleted`) |

---

## Functional Specifications

### SPEC-INDEX-001: Provider Abstraction
**Description**: All search operations MUST be routed through `ISearchProvider`. No consumer code may call a provider SDK directly.  
**Acceptance**: Swapping `provider` in config produces identical query API behaviour.

### SPEC-INDEX-002: Event-Driven Index Writes
**Description**: Product create/update events MUST trigger index document upsert. Product delete events MUST trigger document removal. Index writes MUST be asynchronous and MUST NOT block the product mutation response.  
**Acceptance**: `product.updated` event → provider `replaceDocuments()` called asynchronously. Product API returns before index write completes.

### SPEC-INDEX-003: Index Write Idempotency
**Description**: Indexing the same product multiple times MUST result in a single up-to-date document (no duplicates).  
**Acceptance**: Two consecutive `product.updated` events for the same product → one document in index with latest data.

### SPEC-INDEX-004: Full-Text Search
**Description**: `ISearchProvider.search()` MUST accept a plain-text query and return matching documents ranked by relevance.  
**Acceptance**: Product with title "Medusa T-Shirt" matches query "tshirt" (stemming) and "shirt" (partial). Non-matching products not returned.

### SPEC-INDEX-005: Filtered Search
**Description**: `search()` MUST support filter parameters: `collection_id`, `category_id`, `status`, `tags`.  
**Acceptance**: Search with `filters: { status: ["published"] }` returns no draft products even if text matches.

### SPEC-INDEX-006: Two-Phase Storefront Query
**Description**: `GET /store/products?q=` MUST: (1) get matching product IDs from search index, (2) hydrate full product data via Product module. Product module visibility rules (draft/archived) MUST be applied in phase 2.  
**Acceptance**: Product in index with status `draft` (stale index) → not returned to storefront consumer.

### SPEC-INDEX-007: Bulk Reindex
**Description**: `POST /admin/index/reindex` MUST re-index all products from scratch (page through entire catalogue and rebuild index).  
**Acceptance**: After calling reindex, search returns correct results even after provider wipe.

### SPEC-INDEX-008: Provider Failure Isolation
**Description**: Search provider failures during index writes MUST NOT cause the product mutation to fail. Errors are logged and retried.  
**Acceptance**: Provider throws on `replaceDocuments()`; `PUT /admin/products/:id` still returns 200.

---

## Non-Functional Specifications

### SPEC-INDEX-NFR-001: Index Freshness
**Description**: The search index MUST reflect product changes within 5 seconds of the mutation event.  
**Target**: Event bus delivery + provider write < 5s p95.

### SPEC-INDEX-NFR-002: Query Performance
**Description**: A search query on a 100,000-product catalogue MUST respond in < 100ms (MeiliSearch/Algolia) or < 200ms (built-in).  
**Target**: Measured at p95 under 20 concurrent search requests.

---

## API Contract

### `GET /store/products?q={query}`
| Parameter | Type   | Required | Description               |
|-----------|--------|----------|---------------------------|
| `q`       | string | No       | Full-text search query    |
| `limit`   | number | No       | Max results (default 20)  |
| `offset`  | number | No       | Pagination offset         |

**Response**: `200` — `{ products: Product[], count, limit, offset }`

### `POST /admin/index/reindex`
**Auth**: Admin  
**Response**: `200` — `{ reindexed: number }`

---

## Configuration Schema

```typescript
interface IndexModuleOptions {
  provider?: string            // "built-in" | plugin identifier
  settings?: {
    [indexName: string]: {
      indexSettings?: object   // provider-specific settings
      transformer?: (product: ProductDTO) => object
    }
  }
}
```

---

## Test Checklist

- [ ] `product.created` → document appears in index
- [ ] `product.updated` → document updated in index (no duplicate)
- [ ] `product.deleted` → document removed from index
- [ ] Search by title returns correct products
- [ ] Filter by status excludes drafts
- [ ] Draft product in stale index filtered by phase-2 Product module query
- [ ] Provider throws → product mutation still succeeds
- [ ] Reindex produces correct results from scratch

---

## Dependencies & Interfaces

| Dependency     | Interface Used                     | Direction |
|----------------|------------------------------------|-----------|
| Product module | Product DTO (full relations)       | Outbound  |
| Event Bus      | `product.*` event subscription     | Inbound   |
| ISearchProvider | All search operations             | Outbound  |
| Admin routes   | `POST /admin/index/reindex`        | Inbound   |
| Storefront     | `GET /store/products?q=`           | Inbound   |
