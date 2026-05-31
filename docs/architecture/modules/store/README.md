# Store Module

## Overview

The Store module manages the top-level configuration record for a Medusa installation. It represents the merchant's business identity — the store name, supported currencies, and links to default operational entities such as the default currency, default region, and default sales channel.

A Medusa instance has exactly one store record. This record acts as the root configuration anchor for multi-currency, multi-region setups and is the primary entity managed by the `/admin/stores` API.

## Key Features

- **Single store record**: One row in the `store` table per Medusa deployment.
- **Supported currencies**: A `store_currency` join table tracks which ISO 4217 currencies the store accepts.
- **Default currency**: The currency used when no explicit currency context is provided.
- **Default region & sales channel**: Linked via the Link Modules system (no direct FK); resolved via Query.
- **Store name and metadata**: Displayed in the admin dashboard and used in transactional emails.
- **Admin API**: Full CRUD at `/admin/stores`.
- **Store API**: Public read endpoint at `/store/store` for storefront currency/locale bootstrapping.

## Entities

### Store

| Field           | Type      | Description                            |
|-----------------|-----------|----------------------------------------|
| `id`            | string    | Unique store identifier (`store_*`)    |
| `name`          | string    | Display name of the store              |
| `default_currency_code` | string | ISO 4217 code (e.g., `usd`)    |
| `supported_currencies` | StoreCurrency[] | Accepted currencies         |
| `metadata`      | JSON      | Arbitrary key-value store metadata     |
| `created_at`    | timestamp | Record creation time                   |
| `updated_at`    | timestamp | Last update time                       |

### StoreCurrency

| Field           | Type    | Description                              |
|-----------------|---------|------------------------------------------|
| `id`            | string  | Unique ID                                |
| `store_id`      | string  | FK to store                              |
| `currency_code` | string  | ISO 4217 code                            |
| `is_default`    | boolean | Whether this is the store default        |
| `created_at`    | timestamp | Record creation time                   |

## API Surface

| Method | Path            | Auth   | Description                    |
|--------|-----------------|--------|--------------------------------|
| GET    | `/admin/stores` | Admin  | List/get store configuration   |
| POST   | `/admin/stores` | Admin  | Create store (initial setup)   |
| PUT    | `/admin/stores/:id` | Admin | Update store settings        |
| GET    | `/store/store`  | Public | Get store for storefront init  |

## Cross-Module Links (via Link Modules)

| Link                           | Description                                    |
|--------------------------------|------------------------------------------------|
| `store ↔ default_sales_channel` | The sales channel used when none is specified |
| `store ↔ default_region`        | Fallback region for customer-facing flows     |

These are resolved at query time using `@medusajs/framework`'s `Query` helper:

```typescript
const { data: stores } = await query.graph({
  entity: "store",
  fields: ["id", "name", "default_sales_channel.*"],
})
```

## Module Registration

```typescript
// Automatically registered as a core module — no explicit config required.
import { Modules } from "@medusajs/framework/utils"
// Modules.STORE is available out of the box.
```

## Dependencies

| Dependency           | Purpose                                |
|----------------------|----------------------------------------|
| `@medusajs/framework` | Core module infrastructure            |
| Link Modules         | `store ↔ sales channel`, `store ↔ region` |
| Currency data        | ISO 4217 currency code validation      |
