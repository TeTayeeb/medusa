# @medusajs/fulfillment-manual

Manual fulfillment provider for Medusa v2. Implements `IFulfillmentProvider` via `AbstractFulfillmentProviderService` and registers under `Modules.FULFILLMENT` with identifier `manual`.

This is the **default** fulfillment provider included with Medusa. It has no carrier integration — fulfillments are created as records in the database and must be manually confirmed by a store operator through the Medusa Admin or API.

## Installation

```bash
npm install @medusajs/fulfillment-manual
```

The provider ships as the default fulfillment option and is typically already configured in new Medusa projects.

## Configuration

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/fulfillment",
      options: {
        providers: [
          {
            resolve: "@medusajs/fulfillment-manual",
            id: "manual",
          },
        ],
      },
    },
  ],
})
```

No additional options are required or supported.

## Fulfillment options

The provider exposes two fulfillment option types:

| ID | Description |
|---|---|
| `manual-fulfillment` | Standard outbound shipment |
| `manual-fulfillment-return` | Return shipment (`is_return: true`) |

These options appear in the Medusa Admin under Shipping Options when setting up shipping methods for regions.

## Provider API

### `getFulfillmentOptions()`
Returns:
```json
[
  { "id": "manual-fulfillment" },
  { "id": "manual-fulfillment-return", "is_return": true }
]
```

### `validateOption(data)` → `true`
Always returns `true`. Any option data is considered valid.

### `validateFulfillmentData(optionData, data, context)` → `data`
Pass-through — returns the input `data` unchanged. No external validation is performed.

### `canCalculate()` → `false`
This provider does **not** support dynamic shipping price calculation. Shipping prices must be set statically on the shipping method.

### `calculatePrice(optionData, data, context)` → throws
Throws `Error("Manual fulfillment does not support price calculation")`. This method should never be called because `canCalculate()` returns `false`.

### `createFulfillment()` → `{ data: {}, labels: [] }`
Creates a fulfillment record with no carrier data and no shipping labels. The operator is responsible for physically shipping the order and marking it fulfilled.

### `cancelFulfillment()` → `{}`
No-op — there is no carrier integration to notify. Returns an empty object.

### `createReturnFulfillment()` → `{ data: {}, labels: [] }`
Same as `createFulfillment` but for return shipments.

## Operator workflow

```
1. Order placed by customer
2. Operator reviews order in Medusa Admin
3. Operator manually ships items (external process)
4. Operator marks fulfillment as "shipped" in Admin → triggers tracking events
5. Fulfillment status updated to "shipped" / "delivered" as operator updates it
```

No automated carrier communication, label generation, or tracking updates occur.

## When to use

- Development and testing environments
- Small operations with manual shipping processes
- Use cases where a custom carrier integration is built on top of Medusa and fulfillment is tracked externally
- Pickup-in-store scenarios

For production carrier integrations, use provider packages such as `@medusajs/fulfillment-shippo` or a custom `AbstractFulfillmentProviderService` implementation.
