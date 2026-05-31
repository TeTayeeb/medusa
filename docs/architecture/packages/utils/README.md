# @medusajs/framework/utils

> Version: 2.15.4 · Package path: `packages/core/utils`

The **utils** package is the runtime engine beneath every Medusa module. It ships the `MedusaService` factory, all service decorators, the `MedusaError` hierarchy, ID generation, input validation, and dozens of helpers that every module and API route depends on. If `@medusajs/framework/types` defines *what* data looks like, `@medusajs/framework/utils` defines *how* code behaves.

---

## Installation

```bash
pnpm add @medusajs/framework
```

```typescript
import {
  MedusaError,
  MedusaService,
  InjectManager,
  InjectTransactionManager,
  MedusaContext,
  EmitEvents,
  Modules,
  ContainerRegistrationKeys,
  generateEntityId,
  validateEmail,
} from "@medusajs/framework/utils"
```

---

## Key Exports

### `MedusaError` — typed error class

```typescript
import { MedusaError } from "@medusajs/framework/utils"

// Throw a well-known typed error
throw new MedusaError(
  MedusaError.Types.NOT_FOUND,
  `Product with id: ${id} was not found`
)

// Error types available:
// MedusaError.Types.NOT_FOUND
// MedusaError.Types.INVALID_DATA
// MedusaError.Types.NOT_ALLOWED
// MedusaError.Types.CONFLICT
// MedusaError.Types.UNEXPECTED_STATE
// MedusaError.Types.INVALID_ARGUMENT
// MedusaError.Types.DB_ERROR
```

### `MedusaService` — base class factory for module services

```typescript
import { MedusaService } from "@medusajs/framework/utils"
import { Product, ProductVariant } from "./models"

class MyProductService extends MedusaService({
  Product,
  ProductVariant,
}) {}
// Inherits: list, retrieve, create, update, delete, softDelete, restore
// for both Product and ProductVariant automatically
```

### Service decorators

```typescript
import {
  InjectManager,
  InjectTransactionManager,
  MedusaContext,
  EmitEvents,
} from "@medusajs/framework/utils"
import type { Context } from "@medusajs/framework/types"

class OrderService extends MedusaService({ Order }) {
  // @InjectManager — wraps method in a read-only entity manager
  @InjectManager()
  async getOrder(
    id: string,
    @MedusaContext() sharedContext: Context = {}
  ) {
    return await this.orderService_.retrieve(id, {}, sharedContext)
  }

  // @InjectTransactionManager — wraps method in a write transaction
  @InjectTransactionManager()
  @EmitEvents()
  protected async createOrder_(
    data: CreateOrderDTO,
    @MedusaContext() sharedContext: Context = {}
  ) {
    return await this.orderService_.create(data, sharedContext)
  }
}
```

### `Modules` enum — well-known module keys

```typescript
import { Modules } from "@medusajs/framework/utils"

const productService = container.resolve(Modules.PRODUCT)
const orderService   = container.resolve(Modules.ORDER)
const cartService    = container.resolve(Modules.CART)
const paymentService = container.resolve(Modules.PAYMENT)
// Also: INVENTORY, PRICING, PROMOTION, CUSTOMER, FULFILLMENT, REGION…
```

### `ContainerRegistrationKeys` — DI container well-known keys

```typescript
import { ContainerRegistrationKeys } from "@medusajs/framework/utils"

const query  = container.resolve(ContainerRegistrationKeys.QUERY)
const logger = container.resolve(ContainerRegistrationKeys.LOGGER)
const config = container.resolve(ContainerRegistrationKeys.CONFIG_MODULE)
const featureFlags = container.resolve(ContainerRegistrationKeys.FEATURE_FLAGS)
```

### ID generation

```typescript
import { generateEntityId } from "@medusajs/framework/utils"

// Generates NanoID with a domain prefix, e.g. "prod_01HX7KBCXYZ..."
const productId  = generateEntityId(undefined, "prod")
const variantId  = generateEntityId(undefined, "variant")
const orderId    = generateEntityId(undefined, "order")
```

### Input validators

```typescript
import { validateEmail, validateUrl } from "@medusajs/framework/utils"

validateEmail("not-an-email")  // throws MedusaError.INVALID_DATA
validateUrl("not-a-url")       // throws MedusaError.INVALID_DATA
```

---

## Module Subsystems

| Sub-directory | Purpose |
|---|---|
| `exceptions/` | PostgreSQL error helpers (`isDuplicateError`) |
| `dml/` | Data Modelling Layer helpers |
| `event-bus/` | Event name constants and builder utilities |
| `feature-flags/` | `isFeatureEnabled()` helper |
| `modules-sdk/` | `MedusaService`, decorators, `define-link`, `joiner-config-builder` |
| `common/` | `isObject`, `isString`, `promiseAll`, `MedusaV2Flag` |
| `dal/` | MikroORM utilities, `buildQuery`, `createPgConnection` |
| `link/` | Module link helpers |
| `migrations/` | Migration script runner |
| `translations/` | i18n helpers |

---

## Philosophy

- **Decorator-driven** — cross-cutting concerns (transactions, events, context) are applied via TypeScript decorators, keeping business logic clean.
- **Fail fast** — `MedusaError` provides named error types that map to HTTP status codes at the framework layer.
- **Zero external dependencies for core logic** — ID generation, validation, and decorators have no heavy runtime deps.

---

## See Also

- [`@medusajs/framework/types`](../types/README.md)
- [`@medusajs/modules-sdk`](../modules-sdk/README.md)
