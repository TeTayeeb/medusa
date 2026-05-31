# Locking Module

## Overview

The Locking module provides a distributed locking abstraction that prevents race conditions in concurrent business operations. It is used primarily to serialise access to critical sections such as order processing, inventory reservation, and payment capture — scenarios where two simultaneous requests could otherwise produce inconsistent state.

The module follows the same provider pattern used throughout Medusa: a stable interface abstracts away the underlying locking mechanism, making it trivial to switch between in-process locks (for development) and distributed backends (Redis or PostgreSQL advisory locks) for production deployments.

## Key Features

- **Provider abstraction**: `ILockingModule` is implemented by pluggable provider packages.
- **Built-in providers**: `in-process` (development), `redis` and `postgresql` (production).
- **Simple API**: `acquire()`, `release()`, `execute()` cover all locking patterns.
- **`execute()` convenience**: Acquires a lock, runs a callback, releases on completion or error — atomically.
- **Configurable TTL**: Lock time-to-live prevents deadlocks caused by crashed workers.
- **Retry strategy**: Configurable retry count and backoff for lock acquisition under contention.

## Provider Interface

```typescript
interface ILockingModule {
  acquire(
    keys: string | string[],
    options?: { expire?: number }
  ): Promise<void>

  release(
    keys: string | string[],
    options?: {}
  ): Promise<boolean>

  execute<T>(
    keys: string | string[],
    job: () => Promise<T>,
    options?: { expire?: number }
  ): Promise<T>
}
```

## Usage Example

```typescript
const lockingModule = container.resolve(Modules.LOCKING)

// Wrap critical section
const result = await lockingModule.execute(
  `inventory:reserve:variant_${variantId}`,
  async () => {
    const available = await inventoryModule.retrieveAvailability(variantId)
    if (available < quantity) throw new MedusaError(...)
    await inventoryModule.adjustInventory(variantId, -quantity)
    return { reserved: true }
  },
  { expire: 5000 } // 5 second TTL
)
```

## Module Registration

```typescript
// Redis provider (production)
{
  resolve: "@medusajs/locking-redis",
  options: {
    redisUrl: process.env.REDIS_URL,
    retryCount: 3,
    retryDelay: 200,
  },
}

// In-process (development — default)
{
  resolve: "@medusajs/locking",
}
```

## Lock Key Conventions

| Pattern                             | Used By                        |
|-------------------------------------|--------------------------------|
| `inventory:reserve:{variant_id}`    | Cart item add / checkout       |
| `order:capture:{order_id}`          | Payment capture workflow       |
| `order:fulfill:{order_id}`          | Fulfillment creation           |
| `cart:complete:{cart_id}`           | Complete cart workflow         |

## Deployment Considerations

| Environment | Provider     | Notes                                    |
|-------------|--------------|------------------------------------------|
| Development | `in-process` | Single-process only; no Redis required   |
| Staging     | `postgresql` | Uses PG advisory locks; no extra infra   |
| Production  | `redis`      | Recommended; low latency, cluster-ready  |

## Dependencies

| Dependency           | Purpose                                |
|----------------------|----------------------------------------|
| `@medusajs/framework` | Module container infrastructure       |
| Redis / PostgreSQL   | Backend for distributed lock storage  |
| Inventory module     | Primary consumer (reservation locks)  |
| Order workflows      | Consumer (capture / fulfillment locks)|
