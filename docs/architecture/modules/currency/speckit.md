# Specification — Currency Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                             |
|-----------------|-----------------------------------|
| Module ID       | `Modules.CURRENCY`                |
| Package         | `@medusajs/currency`              |
| Medusa Version  | 2.15.4                            |
| Type            | Infrastructure / Reference Data   |
| Database tables | `currency`                        |
| API surface     | Admin only (read-only)            |

## 2. Functional Requirements

### FR-CUR-01: List Currencies
- **Given** the system is initialized with seeded ISO 4217 currency data
- **When** `listCurrencies(filters?, config?)` is called
- **Then** return an array of `CurrencyDTO` objects matching the filter criteria, respecting `take`/`skip` pagination

### FR-CUR-02: Retrieve Currency by Code
- **Given** a valid ISO 4217 currency code (e.g., `"USD"`)
- **When** `retrieveCurrency(code)` is called
- **Then** return the matching `CurrencyDTO`
- **If** the code does not exist, throw `MedusaError.Types.NOT_FOUND`

### FR-CUR-03: Paginate and Sort
- **Given** a list request with `take`, `skip`, and `order` configuration
- **When** the service processes the request
- **Then** return the correctly paginated and sorted subset of currencies

### FR-CUR-04: Field Selection
- **Given** a list or retrieve request with a `fields` array
- **When** the service processes the request
- **Then** return only the specified fields in each DTO

### FR-CUR-05: Read-Only Enforcement
- **Given** any attempt to POST, PUT, PATCH, or DELETE a currency via the Admin API
- **When** the request is processed
- **Then** the operation returns 404 (no route defined); no write routes are registered

### FR-CUR-06: Seed Data Completeness
- **Given** a fresh Medusa installation
- **When** the database seed runs
- **Then** all active ISO 4217 currencies (approximately 170) are present, including minor currencies and special codes (XAU, XDR, XXX)

## 3. Non-Functional Requirements

| ID          | Requirement                        | Target / Constraint                         |
|-------------|------------------------------------|---------------------------------------------|
| NFR-CUR-01  | List response time                 | < 100ms (warm, no cache)                    |
| NFR-CUR-02  | Retrieve response time             | < 20ms (PK lookup)                          |
| NFR-CUR-03  | Data completeness                  | All ~170 active ISO 4217 currencies present |
| NFR-CUR-04  | Decimal precision accuracy         | `decimal_digits` matches ISO 4217 minor units table |
| NFR-CUR-05  | Idempotent seed                    | Re-running seed must not duplicate records  |
| NFR-CUR-06  | Module replaceability              | Custom module implementing `ICurrencyModuleService` must be drop-in compatible |

## 4. Interface Specification

### GET `/admin/currencies`

| Attribute      | Value                                                       |
|----------------|-------------------------------------------------------------|
| Auth required  | Yes (Admin JWT)                                             |
| Query params   | `limit` (int), `offset` (int), `order` (field:ASC/DESC), `fields` (csv), `q` (search) |
| Response 200   | `{ currencies: CurrencyDTO[], count: number, limit: number, offset: number }` |

### GET `/admin/currencies/:code`

| Attribute      | Value                                              |
|----------------|----------------------------------------------------|
| Auth required  | Yes (Admin JWT)                                    |
| Path param     | `code` — 3-char uppercase ISO 4217 code            |
| Response 200   | `{ currency: CurrencyDTO }`                        |
| Response 404   | `{ type: "not_found", message: "Currency with code: X was not found" }` |

## 5. Data Contracts

### CurrencyDTO

```ts
type CurrencyDTO = {
  code: string           // "USD" — 3-char ISO 4217 uppercase
  name: string           // "US Dollar"
  symbol: string         // "$"
  symbol_native: string  // "$"
  decimal_digits: number // 2 — number of minor units
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### FilterableCurrencyProps

```ts
type FilterableCurrencyProps = {
  code?: string | string[]
  name?: string
}
```

## 6. Seed Data Specification

The module seed must include at minimum:

| Requirement                       | Detail                                         |
|-----------------------------------|------------------------------------------------|
| G20 currencies                    | USD, EUR, GBP, JPY, CNY, INR, BRL, AUD, CAD, ZAR, etc. |
| All EU/EEA currencies             | Including CHF, NOK, SEK, DKK, ISK, etc.        |
| ISO 4217 special codes            | XAU (gold), XAG (silver), XDR (SDR), XXX (no currency) |
| Correct decimal_digits            | Sourced from ISO 4217 minor units table (JPY=0, KWD=3, etc.) |

## 7. Edge Cases

| Case                                      | Expected Behaviour                                         |
|-------------------------------------------|------------------------------------------------------------|
| Code `"JPY"` requested                    | Returns `decimal_digits: 0`                                |
| Code `"KWD"` (Kuwaiti Dinar)              | Returns `decimal_digits: 3`                                |
| Code `"XXX"` (no currency)               | Present in seed, retrievable                               |
| Lowercase `"usd"` in path param           | Normalized to uppercase before lookup                      |
| Soft-deleted currency                     | Excluded from default list; retrievable with `withDeleted` config |
| Currency code with accents or special chars | Rejected by upstream validation (ISO code format only)   |

## 8. Module Boundaries

| In Scope                                    | Out of Scope                                         |
|---------------------------------------------|------------------------------------------------------|
| ISO currency metadata (code, name, symbol)  | Exchange rates or FX conversion                      |
| Read API for listing and retrieving         | Write API (create/update/delete via HTTP)            |
| Decimal precision for monetary arithmetic   | Locale-specific number formatting                    |
| Seed script for ISO 4217 data              | Custom or cryptocurrency definitions (out of box)    |

## 9. Acceptance Criteria Summary

- [ ] `GET /admin/currencies` returns all seeded currencies (≥ 170 records)
- [ ] `GET /admin/currencies/USD` returns `{ code: "USD", decimal_digits: 2, ... }`
- [ ] `GET /admin/currencies/JPY` returns `{ decimal_digits: 0 }`
- [ ] `GET /admin/currencies/INVALID` returns HTTP 404
- [ ] Pagination: `?limit=10&offset=0` returns exactly 10 records
- [ ] Seed is idempotent: running twice produces no duplicates
- [ ] No POST/PUT/DELETE routes exist under `/admin/currencies`
