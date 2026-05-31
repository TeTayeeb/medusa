# SpecKit — Stock Location Module

**Module:** `@medusajs/stock-location`  
**Version:** Medusa v2.15.4  
**Document type:** Functional & Technical Specification

---

## 1. Functional Specifications

### 1.1 Stock Locations

**F-LOC-01: Create Stock Location**
- `name` is required (non-empty string).
- An inline `address` object may be provided; or an existing `address_id` referenced.
- A location may have no address (`address_id` is nullable).
- `metadata` is an optional arbitrary JSON object for custom attributes.

**F-LOC-02: Update Stock Location**
- `name` and `metadata` may be updated.
- `address` may be updated inline (updates the linked `StockLocationAddress`) or replaced by providing a new `address_id`.

**F-LOC-03: Delete Stock Location**
- Soft-deletes the location record.
- Does NOT automatically soft-delete the linked `StockLocationAddress` (addresses may be reused).
- Linked Module Link records (SalesChannel, FulfillmentProvider) should be removed by the calling workflow.

**F-LOC-04: List / Filter Locations**
- Filterable by `name`, `address.city`, `address.country_code`, `address.province`, `address.postal_code`.
- Supports full-text search on `name`, all address fields.
- Paginated via `offset` / `limit`.
- Supports relation expansion: `address`.

**F-LOC-05: Upsert**
- `upsertStockLocations` creates a new location if no matching ID/name found, otherwise updates.
- Used by bulk-import workflows.

### 1.2 Stock Location Addresses

**F-ADDR-01: Create Address**
- `address_1` and `country_code` are required.
- All other fields are optional.

**F-ADDR-02: Upsert Address**
- `upsertStockLocationAddresses` creates or updates the address by ID.
- Address updates propagate to the linked location via relation.

**F-ADDR-03: Address Cascade**
- If a `StockLocationAddress` is deleted, the related `StockLocation` is also deleted (cascade defined in DML model).

---

## 2. Business Rules

| ID | Rule | Enforcement |
|---|---|---|
| BR-01 | `name` is required on `StockLocation` | DB NOT NULL constraint |
| BR-02 | `address_1` and `country_code` required on `StockLocationAddress` | DB NOT NULL + API validation |
| BR-03 | Deleting `StockLocationAddress` cascades to `StockLocation` | DML cascade |
| BR-04 | No uniqueness constraint on location `name` | By design — duplicate names allowed |
| BR-05 | `country_code` must be a valid ISO 3166-1 alpha-2 code | API-layer validation (zod) |
| BR-06 | Location soft-delete does NOT cascade to inventory levels | By design — inventory data preserved |
| BR-07 | Module Link records must be cleaned up by calling workflow | Convention; not enforced by module |

---

## 3. API Contracts

### 3.1 POST /admin/stock-locations

**Request:**
```json
{
  "name": "Main Warehouse",
  "address": {
    "address_1": "123 Commerce Blvd",
    "city": "Portland",
    "country_code": "US",
    "province": "OR",
    "postal_code": "97201"
  },
  "metadata": {
    "wms_id": "WH-001",
    "timezone": "America/Los_Angeles"
  }
}
```

**Response `201`:**
```json
{
  "stock_location": {
    "id": "sloc_01EXAMPLE",
    "name": "Main Warehouse",
    "address_id": "laddr_01EXAMPLE",
    "address": {
      "id": "laddr_01EXAMPLE",
      "address_1": "123 Commerce Blvd",
      "city": "Portland",
      "country_code": "US",
      "province": "OR",
      "postal_code": "97201"
    },
    "metadata": { "wms_id": "WH-001", "timezone": "America/Los_Angeles" },
    "created_at": "2024-01-01T00:00:00.000Z",
    "updated_at": "2024-01-01T00:00:00.000Z"
  }
}
```

### 3.2 GET /admin/stock-locations

**Query params:**
| Param | Type | Description |
|---|---|---|
| `q` | string | Search across `name`, address fields |
| `name` | string | Exact or partial name filter |
| `address.city` | string | Filter by address city |
| `address.country_code` | string | Filter by country |
| `fields` | string | Field projection |
| `expand` | string | Relation expansion (e.g. `address`) |
| `offset` | number | Pagination offset |
| `limit` | number | Page size (default 20, max 100) |

**Response `200`:**
```json
{
  "stock_locations": [
    {
      "id": "sloc_01",
      "name": "Main Warehouse",
      "address": { ... }
    }
  ],
  "count": 1,
  "offset": 0,
  "limit": 20
}
```

### 3.3 POST /admin/stock-locations/:id

**Request (partial update):**
```json
{
  "name": "Main Warehouse — East Wing",
  "address": {
    "address_2": "Unit B"
  }
}
```

**Response `200`:**
```json
{
  "stock_location": {
    "id": "sloc_01",
    "name": "Main Warehouse — East Wing",
    "address": { "id": "laddr_01", "address_2": "Unit B", ... }
  }
}
```

### 3.4 DELETE /admin/stock-locations/:id

**Response `200`:**
```json
{
  "id": "sloc_01",
  "object": "stock_location",
  "deleted": true
}
```

---

## 4. Validation Rules

| Field | Rule |
|---|---|
| `name` | Required, non-empty string, max 255 chars |
| `address_1` | Required on address, non-empty |
| `country_code` | Required on address, ISO 3166-1 alpha-2 (2 chars) |
| `postal_code` | Optional string, max 64 chars |
| `phone` | Optional string, max 64 chars |
| `metadata` | Valid JSON object |

---

## 5. Module Link API (Sales Channel Assignment)

```ts
// Assign location to sales channel
POST /admin/sales-channels/:id/stock-locations
{ "location_id": "sloc_01" }

// Remove location from sales channel
DELETE /admin/sales-channels/:id/stock-locations/:loc_id

// List locations for a sales channel
GET /admin/sales-channels/:id/stock-locations
```

---

## 6. StockLocationDTO

```ts
{
  id: string
  name: string
  address_id: string | null
  address?: {
    id: string
    address_1: string
    address_2: string | null
    company: string | null
    city: string | null
    country_code: string
    phone: string | null
    province: string | null
    postal_code: string | null
    metadata: Record<string, unknown> | null
    created_at: Date
    updated_at: Date
  }
  metadata: Record<string, unknown> | null
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

---

## 7. Integration Test Scenarios

| Scenario | Expected |
|---|---|
| Create location with inline address | Both records created; address_id set |
| Create location without address | `address_id = null` |
| Update location with address inline | Linked address record updated |
| Delete address | Linked location also deleted (cascade) |
| List with `address.country_code=US` | Returns only US locations |
| Soft-delete location | `deleted_at` set; not returned in default list |
| Link location to sales channel | Module Link record created |
| Unlink location from sales channel | Module Link record removed |
