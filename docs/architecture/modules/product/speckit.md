# SpecKit — Product Module

> Medusa v2.15.4 | Functional & Technical Specification

---

## 1. Functional Capabilities

### 1.1 Product Lifecycle Management
- Create products with full attribute set (title, handle, description, status, dimensions, metadata)
- Update any product attribute individually or in bulk
- Soft-delete products with cascade to variants, options, images
- Restore soft-deleted products

### 1.2 Variant Management
- Add, update, remove variants on an existing product
- Each variant is a purchasable SKU with: SKU, barcode, EAN, UPC, HS code, weight, dimensions, `allow_backorder`, `manage_inventory` flags
- Variant position ordering

### 1.3 Option & Option Value Management
- Define configurable dimensions (e.g. "Color", "Size") per product
- Each variant carries exactly one value per option
- Options can be reordered

### 1.4 Taxonomy
- **Collections**: curated groups with handle and metadata
- **Categories**: unlimited-depth tree with `is_active` and `is_internal` flags
- **Tags**: free-form labels, auto-deduped by value
- **Types**: single label per product

### 1.5 Image Management
- Multiple images per product; ordered by `rank`
- Images are URLs (CDN-agnostic); the module stores only the URL string

### 1.6 Import / Export
- CSV-based bulk import via `importProductsWorkflow`
- CSV export with server-sent progress via `exportProductsWorkflow`

---

## 2. Business Rules

| Rule | Description |
|---|---|
| BR-P-001 | `handle` must be unique (case-insensitive) within non-deleted products |
| BR-P-002 | `handle` must match pattern `^[a-z0-9]+(?:-[a-z0-9]+)*$` |
| BR-P-003 | A product must have at least one variant before it can be set to `published` status |
| BR-P-004 | A variant must carry exactly one `ProductOptionValue` per option defined on its parent product |
| BR-P-005 | `ProductCategory.parent_category_id` must not create cycles |
| BR-P-006 | Deleting a `ProductOption` is only allowed if no variants reference its values |
| BR-P-007 | `ProductType` and `ProductTag` are upserted by value (deduplication) |
| BR-P-008 | `is_giftcard` products cannot have `manage_inventory = true` variants |

---

## 3. API Contracts

### POST `/admin/products`

**Request**
```ts
{
  title: string                    // required
  handle?: string                  // auto-derived from title if omitted
  subtitle?: string
  description?: string
  status?: "draft" | "proposed" | "published" | "rejected"
  is_giftcard?: boolean            // default: false
  discountable?: boolean           // default: true
  thumbnail?: string
  weight?: number
  length?: number | null
  height?: number | null
  width?: number | null
  origin_country?: string
  hs_code?: string
  mid_code?: string
  material?: string
  collection_id?: string
  type_id?: string
  tags?: { id?: string; value: string }[]
  categories?: { id: string }[]
  images?: { url: string }[]
  options?: {
    title: string
    values: string[]
  }[]
  variants?: {
    title: string
    sku?: string
    barcode?: string
    ean?: string
    upc?: string
    allow_backorder?: boolean
    manage_inventory?: boolean
    options: Record<string, string>   // { "Color": "Blue", "Size": "XL" }
    prices?: { amount: number; currency_code: string }[]
  }[]
  metadata?: Record<string, unknown>
}
```

**Response** `201 Created`
```ts
{ product: ProductDTO }
```

### GET `/admin/products`

**Query Parameters**
```
q              Full-text search (title, description, variants.title)
status[]       Filter by status
collection_id[]
category_id[]  Supports descendant tree with include_category_children=true
tag_id[]
type_id[]
sales_channel_id[]
limit          Default 20, max 100
offset         Default 0
order          Field + direction, e.g. "-created_at"
fields         Comma-separated field selector
```

**Response** `200 OK`
```ts
{
  products: ProductDTO[]
  count: number
  offset: number
  limit: number
}
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `title` | Required string, min 1 char, max 255 chars |
| `handle` | Optional; if provided must match slug regex; must be globally unique |
| `status` | Must be one of `draft`, `proposed`, `published`, `rejected` |
| `weight`, `length`, `height`, `width` | Non-negative float or null |
| `hs_code` | Max 20 chars |
| `currency_code` (prices) | Must be a valid ISO 4217 code; normalized to lowercase |
| `amount` (prices) | Non-negative integer (in smallest currency unit) |
| `options[].title` | Required; unique within product |
| `variants[].options` | Must have a key for every option defined on the product |
| `category_id` | Must reference existing, non-deleted category |

---

## 5. Performance Considerations

| Concern | Mitigation |
|---|---|
| Large catalog list queries | Partial indexes on `status`, `collection_id`, `type_id`; cursor-based pagination |
| Category tree traversal | Recursive CTE or in-memory tree build; result cached at `remoteQuery` level |
| Full-text search | `searchable()` fields indexed; fallback to ILIKE for simple queries |
| Bulk import | `importProductsWorkflow` processes batches of 100; uses `upsert` semantics |
| Event storm prevention | `MessageAggregator` batches events; single flush per transaction |
| N+1 on variants/options | `FindConfig.relations` drives join strategy; eager loading controlled per endpoint |
