# Software Design Document — Index (Search) Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa Index module (v2.15.4). It covers the provider interface, event subscription architecture, document transformation pipeline, query flow, and integration with the storefront product API.

## 2. Architecture Overview

The Index module is a reactive system: it subscribes to Product module events and maintains a search index in sync with the product catalogue. The search query path is entirely separate from the index write path.

```
Write Path:
Product Module Event Bus
  → product.created / product.updated / product.deleted
      → IndexSubscriber (IEventBusModuleService subscriber)
          → ProductDocumentTransformer.transform(product)
              → ISearchProvider.addDocuments / replaceDocuments / deleteDocument

Query Path:
GET /store/products?q=...
  → ProductSearchWorkflow
      → searchProductsStep
          → ISearchProvider.search(index, query, options)
              → { hits, count, offset }
```

## 3. Provider Interface Detail

```typescript
interface ISearchProvider {
  createIndex(indexName: string, options?: IndexOptions): Promise<void>
  getIndex(indexName: string): Promise<IndexInfo>
  addDocuments(indexName: string, documents: Document[]): Promise<IndexResponse>
  replaceDocuments(indexName: string, documents: Document[]): Promise<IndexResponse>
  deleteDocument(indexName: string, id: string): Promise<void>
  deleteAllDocuments(indexName: string): Promise<void>
  search(indexName: string, query: string, options?: SearchOptions): Promise<SearchResult>
  updateSettings(indexName: string, settings: object): Promise<void>
}
```

## 4. Event Subscription Design

The `IndexSubscriber` class registers handlers for product lifecycle events using the `@OnEvent` decorator pattern:

```typescript
@Injectable()
export class IndexSubscriber {
  @OnEvent([ProductEvents.PRODUCT_CREATED, ProductEvents.PRODUCT_UPDATED])
  async handleProductUpsert({ data }: { data: { id: string } }) {
    const product = await this.productService.retrieve(data.id, {
      relations: ["variants", "tags", "categories"],
    })
    const doc = this.transformer.transform(product)
    await this.searchProvider.replaceDocuments("products", [doc])
  }

  @OnEvent(ProductEvents.PRODUCT_DELETED)
  async handleProductDelete({ data }: { data: { id: string } }) {
    await this.searchProvider.deleteDocument("products", data.id)
  }
}
```

Bulk re-indexing (e.g., on initial setup or provider switch) is handled by `IndexModuleService.reindex()`, which pages through all products and batch-inserts documents.

## 5. Document Transformation

The `ProductDocumentTransformer` maps a `ProductDTO` to a flat, searchable document:

```typescript
transform(product: ProductDTO): Document {
  return {
    id:           product.id,
    title:        product.title,
    description:  product.description ?? "",
    handle:       product.handle,
    status:       product.status,
    tags:         product.tags?.map(t => t.value) ?? [],
    category_ids: product.categories?.map(c => c.id) ?? [],
    collection_id:product.collection_id,
    variant_titles: product.variants?.map(v => v.title) ?? [],
    skus:           product.variants?.map(v => v.sku).filter(Boolean) ?? [],
    thumbnail:      product.thumbnail,
    updated_at:     product.updated_at,
  }
}
```

The transformer is extensible: custom providers can override it to add or remove fields before indexing.

## 6. Built-in Index Provider Design

The built-in provider stores documents in a `search_document` table:

| Field        | Type    | Description                             |
|--------------|---------|-----------------------------------------|
| `id`         | string  | Document ID (product ID)                |
| `index_name` | string  | Index identifier (e.g., `products`)     |
| `document`   | JSONB   | Full document JSON                      |
| `search_vector` | tsvector | PostgreSQL full-text search vector |
| `updated_at` | timestamp | Last index update                     |

Searches use `plainto_tsquery` and GIN index on `search_vector`. Suitable for catalogues up to ~50,000 products.

## 7. Search Options

```typescript
interface SearchOptions {
  limit?: number       // default 20
  offset?: number      // default 0
  filters?: {
    collection_id?: string[]
    category_id?: string[]
    status?: string[]
    tags?: string[]
  }
  sort?: string        // e.g., "updated_at:desc"
  highlight?: boolean  // wrap matched terms in <em>
}
```

## 8. Storefront API Integration

`GET /store/products?q=search+term` triggers:

1. Request validation (strip unsafe filters).
2. `useQueryGraphStep` to retrieve product IDs from the search provider.
3. A second `useQueryGraphStep` to hydrate full product data from the Product module using the returned IDs.
4. Response assembly with standard pagination headers.

This two-phase approach ensures that ACL and visibility rules (e.g., draft products) from the Product module are always applied even when IDs come from the search index.

## 9. Reindex Workflow

```typescript
export const reindexProductsWorkflow = createWorkflow(
  "reindex-products",
  () => {
    const products = listAllProductsStep()
    const documents = transformProductsStep(products)
    replaceAllDocumentsStep(documents)
  }
)
```

Triggered via `POST /admin/index/reindex` or on first-time provider setup.

## 10. Error Handling

| Scenario                    | Behaviour                                          |
|-----------------------------|----------------------------------------------------|
| Provider unavailable        | Log error; queue event for retry (3 attempts)      |
| Product not found for index | Skip silently; log warning                         |
| Index not found             | Auto-create index on first document insert         |
| Search timeout              | Return empty results with `503` status             |
