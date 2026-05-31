# SpecKit — `@medusajs/core-flows`

**Version:** 2.15.4  
**Spec Type:** Behavioural + Interface Specification

---

## 1. Package Identity

| Field | Value |
|---|---|
| Package name | `@medusajs/core-flows` |
| NPM scope | `@medusajs` |
| Source path | `packages/core/core-flows` |
| Main entry | `dist/index.js` |
| Types entry | `dist/index.d.ts` |

---

## 2. Workflow Interface Contract

Every workflow exported from `@medusajs/core-flows` conforms to `ReturnWorkflow<TInput, TOutput, THooks>`:

```typescript
// Invocation pattern
const { result, transaction, errors } = await someWorkflow(scope).run({
  input: { … },        // typed to workflow's TInput
  idempotencyKey?: string,
  resultFrom?: string, // step ID to read result from (default: last step)
  throwOnError?: boolean, // default: true
})

// Sub-workflow pattern
const workflowOutput = someWorkflow.runAsStep({ input })
```

### `run()` Return Value

```typescript
type WorkflowRunResult<TOutput> = {
  result: TOutput                   // resolved output of workflow
  transaction: DistributedTransaction // orchestration state
  errors: TransactionStepError[]    // empty on success
}
```

---

## 3. Workflow Catalogue (Condensed)

### Cart Workflows

```typescript
// createCartsWorkflow
input:  { cartsData: CreateCartDTO[] }
output: CartDTO[]
hooks:  —

// addToCartWorkflow
input:  { id: string; items: AddToCartDTO[] }
output: CartDTO
hooks:  —

// completeCartWorkflow
input:  { id: string; idempotencyKey?: string }
output: OrderDTO
hooks:  cartCompleted({ order: OrderDTO })

// updateCartWorkflow
input:  { id: string } & UpdateCartDTO
output: CartDTO
hooks:  —
```

### Order Workflows

```typescript
// createOrderWorkflow
input:  { cart_id: string }
output: OrderDTO
hooks:  orderCreated({ order: OrderDTO })

// cancelOrderWorkflow
input:  { order_id: string }
output: OrderDTO
hooks:  orderCanceled({ order: OrderDTO })

// createReturnWorkflow
input:  CreateOrderReturnDTO
output: ReturnDTO
hooks:  returnCreated({ returnOrder: ReturnDTO })

// createExchangeWorkflow
input:  CreateOrderExchangeDTO
output: ExchangeDTO
hooks:  exchangeCreated({ exchange: ExchangeDTO })
```

### Product Workflows

```typescript
// createProductsWorkflow
input:  { productsData: CreateProductDTO[] }
output: ProductDTO[]
hooks:  productsCreated({ products: ProductDTO[] })

// updateProductsWorkflow
input:  { selector: FilterableProductProps; update: UpdateProductDTO }
output: ProductDTO[]
hooks:  productsUpdated({ products: ProductDTO[] })

// importProductsWorkflow
input:  { fileKey: string }
output: { summary: { toCreate: number; toUpdate: number } }
hooks:  —
```

---

## 4. Step Interface Contract

```typescript
// Step invocation (direct call inside createWorkflow composer)
const stepOutput: WorkflowData<TOutput> = someStep(stepInput)

// Step type signature
type StepFunction<TInput, TOutput> = (
  input: TInput | WorkflowData<TInput>
) => WorkflowData<TOutput>
```

---

## 5. Invariants

1. **Compensation idempotency:** All compensation functions MUST be safe to call multiple times. A second call for the same data MUST NOT throw.
2. **No direct DB access:** Steps MUST NOT import or instantiate ORM entities. All persistence goes through module services.
3. **Workflow ID uniqueness:** Registering two workflows with the same ID will cause the second to overwrite the first at runtime. This MUST be treated as a configuration error.
4. **Hook execution order:** Multiple handlers registered on the same hook execute in registration order. Handlers are independent — failure in one DOES NOT prevent others from running.
5. **Sub-workflow compensation:** When a sub-workflow fails, its compensation runs before the parent workflow's compensation continues up the stack.

---

## 6. Domain-Specific Step Catalogue

### Inventory Steps

```typescript
confirmInventoryStep(input: {
  inventoryItems: {
    inventory_item_id: string
    required_quantity: number
    allow_backorder: boolean
    location_ids: string[]
  }[]
})
// Throws MedusaError.Types.NOT_ALLOWED if any item is out of stock

reserveInventoryStep(input: {
  items: CreateReservationItemInput[]
})
// Returns StepResponse<ReservationItemDTO[], string[]> (IDs for compensation)
```

### Payment Steps

```typescript
authorizePaymentSessionStep(input: {
  id: string
  context: Record<string, unknown>
})
// Returns: PaymentSessionDTO with status "authorized" or throws

capturePaymentStep(input: {
  payment_id: string
  amount?: number  // defaults to full authorized amount
})
// Returns: PaymentDTO with status "captured"
```

---

## 7. Error Behaviour

| Scenario | Behaviour |
|---|---|
| Step throws `MedusaError` | Compensation chain executes; error surfaced to caller |
| Step throws generic `Error` | Wrapped in `MedusaError.Types.UNEXPECTED_STATE`; compensation executes |
| Compensation throws | Logged as `ERROR`; remaining compensations continue |
| Hook handler throws | Logged as `ERROR`; other hook handlers continue; workflow result unaffected |
| `idempotencyKey` already used | Returns cached result; no steps re-executed |

---

## 8. Compatibility Matrix

| Core-flows version | Workflows-SDK version | Framework version |
|---|---|---|
| 2.15.4 | 2.15.4 | 2.15.4 |
| 2.14.x | 2.14.x | 2.14.x |

Core-flows, workflows-sdk, and framework versions MUST be kept in sync within a Medusa installation.
