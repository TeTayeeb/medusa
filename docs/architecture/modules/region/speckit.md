# Specification — Region Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                                         |
|-----------------|-----------------------------------------------|
| Module ID       | `Modules.REGION`                              |
| Package         | `@medusajs/region`                            |
| Medusa Version  | 2.15.4                                        |
| Type            | Commerce Configuration                        |
| Database tables | `region`, `region_country`                    |
| API surface     | Admin (CRUD) + Store (read-only)              |

## 2. Functional Requirements

### FR-REG-01: Create Region
- **Given** a valid `name` and `currency_code` (referencing an existing ISO 4217 currency)
- **When** `POST /admin/regions` is called
- **Then** a new Region record is created and returned

### FR-REG-02: Assign Countries to Region
- **Given** an existing Region and a list of ISO 3166-1 alpha-2 country codes
- **When** `upsertRegionCountries()` is called
- **Then** countries are associated with the region
- **If** a country is already assigned to another region, the assignment is moved (or rejected, based on configuration)

### FR-REG-03: Update Region
- **Given** an existing Region ID and new field values
- **When** `POST /admin/regions/:id` is called
- **Then** the region is updated; unchanged fields are preserved

### FR-REG-04: Delete Region
- **Given** an existing Region ID
- **When** `DELETE /admin/regions/:id` is called
- **Then** the region is soft-deleted; countries are dissociated

### FR-REG-05: List Regions (Admin)
- **Given** an authenticated admin request
- **When** `GET /admin/regions` is called
- **Then** return paginated list of regions with optional country relations

### FR-REG-06: List Regions (Store)
- **Given** an unauthenticated storefront request
- **When** `GET /store/regions` is called
- **Then** return a list of active (non-deleted) regions

### FR-REG-07: Automatic Tax Flag
- **Given** a region with `automatic_taxes: true`
- **When** a cart is created in that region
- **Then** the Cart module automatically triggers tax calculation at checkout

### FR-REG-08: Currency Validation
- **Given** a `currency_code` value
- **When** a region is created or updated
- **Then** the code is validated against the Currency module; invalid codes return 400

## 3. Non-Functional Requirements

| ID          | Requirement                    | Target                               |
|-------------|--------------------------------|--------------------------------------|
| NFR-REG-01  | List response time             | < 150ms with countries relation      |
| NFR-REG-02  | Country uniqueness             | One country per region (enforced)    |
| NFR-REG-03  | Store API availability         | Public read; no auth required        |
| NFR-REG-04  | Soft delete safety             | Orders/carts retain region_id after delete |

## 4. Interface Specification

### POST `/admin/regions`

| Attribute     | Value                                                                           |
|---------------|---------------------------------------------------------------------------------|
| Auth required | Yes (Admin JWT)                                                                 |
| Body          | `{ name: string, currency_code: string, automatic_taxes?: boolean, tax_rate?: number, countries?: string[] }` |
| Response 200  | `{ region: RegionDTO }`                                                         |
| Response 400  | Invalid currency_code or malformed body                                         |

### GET `/store/regions`

| Attribute     | Value                                            |
|---------------|--------------------------------------------------|
| Auth required | No                                               |
| Query params  | `limit`, `offset`, `fields`                      |
| Response 200  | `{ regions: RegionDTO[], count, limit, offset }` |

## 5. Data Contracts

### RegionDTO

```ts
type RegionDTO = {
  id: string
  name: string
  currency_code: string
  automatic_taxes: boolean
  tax_rate: number | null
  tax_code: string | null
  countries: RegionCountryDTO[]
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### RegionCountryDTO

```ts
type RegionCountryDTO = {
  id: string
  iso_2: string       // "DE"
  iso_3: string       // "DEU"
  num_code: string    // "276"
  name: string        // "Germany"
  display_name: string // "Germany"
  region_id: string
}
```

### CreateRegionDTO

```ts
type CreateRegionDTO = {
  name: string
  currency_code: string
  automatic_taxes?: boolean  // default: true
  tax_rate?: number          // 0–100
  tax_code?: string
  countries?: string[]       // Array of ISO 3166-1 alpha-2 codes
  metadata?: Record<string, unknown>
}
```

## 6. Validation Rules

| Field            | Rule                                                                         |
|------------------|------------------------------------------------------------------------------|
| `currency_code`  | Must exist in Currency module; case-insensitive, normalised to uppercase     |
| `tax_rate`       | Float between 0 and 100 (inclusive), if provided                             |
| `countries`      | Each entry must be a valid 2-letter ISO 3166-1 alpha-2 code                  |
| `name`           | Non-empty string, max 255 characters                                         |

## 7. Edge Cases

| Case                                         | Expected Behaviour                                              |
|----------------------------------------------|-----------------------------------------------------------------|
| Delete region that has active carts          | Allowed; carts retain `region_id` as a snapshot reference      |
| Move country from Region A to Region B       | Remove from A first, then assign to B; or upsert handles it    |
| Region with no countries                     | Valid; countries are optional                                   |
| `automatic_taxes: false` + `tax_rate` set    | Manual tax rate used; no automatic tax calculation triggered    |
| Update `currency_code` on existing region    | Allowed; existing carts unaffected (they snapshot on creation) |
| Create two regions with same name            | Allowed (name uniqueness not enforced)                          |

## 8. Module Boundaries

| In Scope                               | Out of Scope                                               |
|----------------------------------------|------------------------------------------------------------|
| Region and country management          | Tax rate rules (Tax module)                                |
| Currency association per region        | Payment provider configuration per region (module links)  |
| Default tax rate on region             | Pricing and price lists                                    |
| Storefront region selection            | Fulfillment provider scoping (module links)                |

## 9. Acceptance Criteria Summary

- [ ] `POST /admin/regions` with valid body creates region and returns it
- [ ] `POST /admin/regions` with invalid `currency_code` returns 400
- [ ] Countries assigned to a region appear in GET response under `countries`
- [ ] `GET /store/regions` returns regions without admin auth
- [ ] Soft-deleted regions do not appear in default list queries
- [ ] `automatic_taxes: false` is stored and returned correctly
- [ ] `DELETE /admin/regions/:id` soft-deletes; region not returned in subsequent list
