# Index (Search) Module

## Overview

The Index module provides an abstraction layer for full-text search and filtered product discovery within Medusa. It decouples the application's search requirements from specific search engine implementations, allowing operators to swap between a built-in basic index and production-grade engines like Algolia or MeiliSearch via plugins.

The module indexes product data in near-real-time by subscribing to product lifecycle events (create, update, delete) emitted by the Product module's event bus integration.

## Key Features

- **Provider abstraction**: A uniform `ISearchProvider` interface is implemented by all search backends.
- **Built-in provider**: A lightweight in-process index for development and small catalogues.
- **Plugin providers**: Algolia (`@medusajs/medusa-plugin-algolia`) and MeiliSearch (`@medusajs/medusa-plugin-meilisearch`) are first-class supported backends.
- **Event-driven indexing**: Product create/update/delete events trigger automatic index synchronisation.
- **Query interface**: Supports full-text queries combined with attribute filters, pagination, and sorting.
- **Storefront integration**: Powers the `?q=` query parameter on `GET /store/products`.

## Provider Interface

```typescript
interface ISearchProvider {
  createIndex(indexName: string, options?: object): Promise<void>
  addDocuments(indexName: string, documents: object[]): Promise<void>
  replaceDocuments(indexName: string, documents: object[]): Promise<void>
  deleteDocument(indexName: string, id: string): Promise<void>
  deleteAllDocuments(indexName: string): Promise<void>
  search(indexName: string, query: string, options?: SearchOptions): Promise<SearchResult>
}
```

## Indexing Flow

```
Product Event Bus
  → product.created / product.updated / product.deleted
      → IndexSubscriber
          → ISearchProvider.addDocuments / replaceDocuments / deleteDocument
              → Search Engine (Algolia / MeiliSearch / built-in)
```

## Indexed Document Shape (Products)

```typescript
{
  id: "prod_01",
  title: "Medusa T-Shirt",
  description: "Comfortable cotton t-shirt...",
  handle: "medusa-t-shirt",
  status: "published",
  tags: ["apparel", "cotton"],
  variants: [{ id: "...", title: "S / Black", sku: "MED-TS-S-BLK" }],
  collection_id: "pcol_01",
  category_ids: ["pcat_01"],
}
```

## Query API

The storefront endpoint accepts:

| Parameter | Type   | Description                              |
|-----------|--------|------------------------------------------|
| `q`       | string | Full-text search query                   |
| `limit`   | number | Max results (default 20)                 |
| `offset`  | number | Pagination offset                        |
| `filters` | object | Attribute filters (collection, category) |

## Module Registration

```typescript
// Example with MeiliSearch
{
  resolve: "@medusajs/medusa-plugin-meilisearch",
  options: {
    config: {
      host: process.env.MEILISEARCH_HOST,
      apiKey: process.env.MEILISEARCH_API_KEY,
    },
    settings: {
      products: {
        indexSettings: { searchableAttributes: ["title", "description", "sku"] },
      },
    },
  },
}
```

## Dependencies

| Dependency           | Purpose                               |
|----------------------|---------------------------------------|
| `@medusajs/framework` | Event bus subscription, container    |
| Product module       | Source of truth for indexed data      |
| Event bus            | Product lifecycle event delivery      |
