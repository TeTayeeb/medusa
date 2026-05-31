# Software Design Document — @medusajs/fulfillment-manual

## 1. Purpose

Provide a no-op fulfillment provider that allows Medusa to create and track fulfillments as database records without any carrier API integration. Serves as the default provider, enabling order management workflows to function end-to-end in development and for operators who manage shipping manually.

## 2. Architecture

```
Modules.FULFILLMENT
  └── ModuleProvider (fulfillment-manual)
        └── ManualFulfillmentService (AbstractFulfillmentProviderService)
              ├── getFulfillmentOptions()             → option descriptors
              ├── validateOption(data)                → true
              ├── validateFulfillmentData(...)        → passthrough
              ├── canCalculate()                      → false
              ├── calculatePrice(...)                 → throws
              ├── createFulfillment()                 → { data: {}, labels: [] }
              ├── cancelFulfillment()                 → {}
              └── createReturnFulfillment()           → { data: {}, labels: [] }
```

## 3. Fulfillment lifecycle

The Fulfillment Module calls provider methods at different stages of the order lifecycle:

```
1. Setup (Admin):
   getFulfillmentOptions()
   → Admin displays shipping options to configure shipping methods

2. Checkout:
   validateOption(data)         → true (any option accepted)
   validateFulfillmentData(...) → data (passthrough)
   canCalculate()               → false (no dynamic pricing)

3. Fulfillment creation (Post-order):
   createFulfillment()          → { data: {}, labels: [] }
   → Fulfillment record created in DB with empty carrier data

4. Cancellation:
   cancelFulfillment()          → {} (no-op)

5. Return:
   createReturnFulfillment()    → { data: {}, labels: [] }
```

## 4. `createFulfillment` design

```ts
async createFulfillment() {
  // No data is being sent anywhere
  return {
    data: {},     // Carrier data (tracking numbers, etc.) — empty
    labels: [],   // Shipping labels — none
  }
}
```

The Fulfillment Module stores `data` in the `Fulfillment.data` field. For the manual provider, this is always an empty object. Status transitions (shipped, delivered) are driven by explicit API/Admin updates, not by webhook callbacks from a carrier.

## 5. Price calculation

```ts
async canCalculate() { return false }

async calculatePrice(optionData, data, context) {
  throw new Error("Manual fulfillment does not support price calculation")
}
```

The Fulfillment Module checks `canCalculate()` before calling `calculatePrice()`. Since `canCalculate()` returns `false`, `calculatePrice()` is never invoked in normal operation. The guard throw exists as a safety net for incorrect integrations.

Shipping prices must be configured statically in the Medusa Admin (fixed price per shipping method).

## 6. Option descriptors

```ts
[
  { id: "manual-fulfillment" },
  { id: "manual-fulfillment-return", is_return: true }
]
```

The `is_return: true` flag tells the Fulfillment Module that the second option should be used for return shipments. Having a dedicated return option allows operators to configure different shipping methods (and prices) for outbound vs. return shipments.

## 7. `validateOption` and `validateFulfillmentData`

Both are pass-through implementations:
```ts
async validateOption(data)                          { return true }
async validateFulfillmentData(optionData, data, _)  { return data }
```

No validation logic is needed since there is no external carrier system to validate against. The Fulfillment Module's own validation handles data integrity at the module level.

## 8. Intentional simplicity

The manual provider is deliberately minimal. It:
- Has no constructor logic (calls `super()` only)
- Has no configuration options
- Has no external I/O
- Has no state

This makes it a reliable baseline and a minimal reference implementation of `AbstractFulfillmentProviderService`.

## 9. Extension pattern

Operators needing basic carrier information (e.g. tracking numbers entered manually) can create a thin extension:

```ts
class MyManualFulfillmentService extends ManualFulfillmentService {
  async createFulfillment(data, items, order, fulfillment) {
    return {
      data: { tracking_number: data.tracking_number },
      labels: [],
    }
  }
}
```

## 10. Dependencies

| Package | Purpose |
|---|---|
| `@medusajs/framework` | `AbstractFulfillmentProviderService` |

No third-party dependencies.
