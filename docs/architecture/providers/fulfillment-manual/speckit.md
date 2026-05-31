# SpecKit — @medusajs/fulfillment-manual

---

## 1. Unit specs — `getFulfillmentOptions`

| # | Scenario | Expected outcome |
|---|---|---|
| U1 | Returns both options | Array with 2 elements |
| U2 | Standard option structure | `{ id: "manual-fulfillment" }` |
| U3 | Return option structure | `{ id: "manual-fulfillment-return", is_return: true }` |

---

## 2. Unit specs — `validateOption`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U4 | Any data | `{}` | Returns `true` |
| U5 | Null data | `null` | Returns `true` |
| U6 | Non-empty data | `{ some_key: "value" }` | Returns `true` |

---

## 3. Unit specs — `validateFulfillmentData`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U7 | Pass-through | `optionData: {}, data: { key: "val" }, context: {}` | Returns `{ key: "val" }` unchanged |
| U8 | Empty data | `data: {}` | Returns `{}` |

---

## 4. Unit specs — `canCalculate`

| # | Scenario | Expected outcome |
|---|---|---|
| U9 | Always false | `canCalculate()` | Returns `false` |

---

## 5. Unit specs — `calculatePrice`

| # | Scenario | Expected outcome |
|---|---|---|
| U10 | Called directly | Any args | Throws `Error("Manual fulfillment does not support price calculation")` |

---

## 6. Unit specs — `createFulfillment`

| # | Scenario | Expected outcome |
|---|---|---|
| U11 | Returns correct structure | Called with no args | Returns `{ data: {}, labels: [] }` |
| U12 | `data` is empty object | — | `result.data` deep-equals `{}` |
| U13 | `labels` is empty array | — | `result.labels` deep-equals `[]` |

---

## 7. Unit specs — `cancelFulfillment`

| # | Scenario | Expected outcome |
|---|---|---|
| U14 | Returns empty object | Called with any args | Returns `{}` |
| U15 | No error thrown | — | Resolves without exception |

---

## 8. Unit specs — `createReturnFulfillment`

| # | Scenario | Expected outcome |
|---|---|---|
| U16 | Returns correct structure | Called with no args | Returns `{ data: {}, labels: [] }` |
| U17 | Same structure as `createFulfillment` | — | Identical return shape |

---

## 9. Unit specs — identifier

| # | Scenario | Expected outcome |
|---|---|---|
| U18 | Static identifier | `ManualFulfillmentService.identifier` | Equals `"manual"` |

---

## 10. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Create order → create fulfillment | Fulfillment record created with `data: {}`, `labels: []` |
| I2 | Create fulfillment → cancel fulfillment | Fulfillment status updated without error |
| I3 | Return order → create return fulfillment | Return fulfillment record created |
| I4 | Shipping option setup in Admin | Both `manual-fulfillment` options visible |
| I5 | Price calculation attempted | `canCalculate()` returns `false`; `calculatePrice` not called by module |

---

## 11. Acceptance criteria

- `canCalculate()` always returns `false`.
- `calculatePrice` always throws when called directly.
- `createFulfillment` and `createReturnFulfillment` always return `{ data: {}, labels: [] }`.
- `cancelFulfillment` always returns `{}` without error.
- Provider registers with `identifier = "manual"`.
- Both fulfillment option IDs are exposed by `getFulfillmentOptions`.
