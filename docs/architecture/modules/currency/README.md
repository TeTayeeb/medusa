# Currency Module

The Currency module (`@medusajs/currency`) is a core Medusa v2 infrastructure module that provides an authoritative, read-only reference of ISO 4217 world currencies. It acts as a shared foundation for the Pricing, Region, and Store modules — any component that needs to denominate a monetary amount or represent a locale-specific currency references this module.

## Purpose

Rather than each module maintaining its own copy of currency metadata, Medusa centralizes currency definitions in a single module. This ensures consistent decimal precision, symbols, and naming across the entire platform.

## Key Features

- **ISO 4217 Reference Data** — Ships with a comprehensive seed of world currencies including code, name, symbol, native symbol, and decimal digit count.
- **Read-Only Design** — Currencies are not created or deleted via API; data is seeded at boot time and treated as immutable reference data.
- **Decimal Precision** — Tracks the number of minor units per currency (e.g., USD = 2, JPY = 0, KWD = 3), critical for correct price arithmetic.
- **Symbol Variants** — Stores both the international symbol (`$`) and the native symbol (`$`) for display flexibility.
- **Lightweight and Cacheable** — Suitable for aggressive caching since data changes rarely; the full dataset (~170 records) fits comfortably in memory.

## Entities

| Entity     | Key Fields                                                                 |
|------------|----------------------------------------------------------------------------|
| `Currency` | `code` (PK, ISO 4217), `name`, `symbol`, `symbol_native`, `decimal_digits` |

## Admin API

| Method | Endpoint                   | Description                        |
|--------|----------------------------|------------------------------------|
| GET    | `/admin/currencies`        | List currencies with pagination    |
| GET    | `/admin/currencies/:code`  | Retrieve a specific currency       |

## Store API

No direct Store API endpoints. Currency data is embedded in Region, Store, and pricing responses.

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.CURRENCY
```

## Service Usage

```ts
const currencyService = container.resolve(Modules.CURRENCY)

// List all currencies
const currencies = await currencyService.listCurrencies()

// Retrieve a specific currency
const usd = await currencyService.retrieveCurrency("USD")

// List with filters and pagination
const [results, count] = await currencyService.listAndCountCurrencies(
  {},
  { take: 20, skip: 0, order: { code: "ASC" } }
)
```

## Data Shape

```ts
type CurrencyDTO = {
  code: string           // ISO 4217 code, e.g. "USD"
  name: string           // Human-readable name, e.g. "US Dollar"
  symbol: string         // e.g. "$"
  symbol_native: string  // Native symbol, e.g. "$"
  decimal_digits: number // Minor unit precision, e.g. 2
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

## Integration Points

| Module   | How It Uses Currency                                              |
|----------|-------------------------------------------------------------------|
| Region   | Each region stores a `currency_code` referencing this module      |
| Pricing  | Money amounts use `currency_code` from this module                |
| Store    | Stores carry a `default_currency_code` and list of supported ones |

## Module Registration

The module is bundled with the default Medusa installation. Explicit registration is only needed in custom setups:

```ts
// medusa-config.ts
export default defineConfig({
  modules: [{ resolve: "@medusajs/currency" }]
})
```

## Version

Medusa v2.15.4
