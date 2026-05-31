# Event Bus Local Module

## Overview

The `event-bus-local` module is Medusa's in-process event bus implementation, designed for development, testing, and single-process deployments. It leverages Node.js's built-in `EventEmitter` under the hood to provide a simple, zero-dependency, synchronous-style publish/subscribe mechanism within a single Node.js process.

## Purpose

In a standard Medusa installation, commerce modules (order, product, cart, etc.) communicate asynchronously through domain events. The local event bus fulfills this contract using the `IEventBusModuleService` interface without requiring any external infrastructure. This makes it trivial to run Medusa locally with no Redis or message-broker dependency.

## Key Features

- **In-process delivery** — events are dispatched entirely within the running Node.js process via `EventEmitter`.
- **Synchronous-style publishing** — `emit()` resolves once all registered handlers for the event have been invoked; no network round-trip.
- **Event grouping** — supports transactional event batching via `EventBusUtils.groupEventsByQueue`, allowing a set of events to be released atomically at the end of a database transaction.
- **Zero external dependencies** — no Redis, no broker, no additional Docker containers needed.
- **Drop-in replacement** — satisfies the same `IEventBusModuleService` interface as `event-bus-redis`, enabling transparent swap-out in production.

## Interface

```typescript
interface IEventBusModuleService {
  emit<T>(eventName: string, data: T, options?: Record<string, unknown>): Promise<void>
  emit<T>(events: { eventName: string; data: T; options?: Record<string, unknown> }[]): Promise<void>
  subscribe(eventName: string, subscriber: Subscriber, context?: SubscriberContext): this
  unsubscribe(eventName: string, subscriber: Subscriber, context?: SubscriberContext): this
}
```

## Configuration

The module requires no configuration. It is registered in `medusa-config.ts` as:

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/event-bus-local",
      key: Modules.EVENT_BUS,
    },
  ],
})
```

This is the default for `NODE_ENV=development` if no event bus module is explicitly provided.

## Limitations

| Constraint | Detail |
|---|---|
| **No persistence** | Events not yet processed are lost if the process crashes or restarts. |
| **Single process only** | Events emitted by one process are invisible to other processes. |
| **No retry** | Failed subscriber invocations are not automatically retried. |
| **No dead-letter queue** | There is no facility to capture or replay failed events. |

## When to Use

| Scenario | Recommendation |
|---|---|
| Local development | ✅ Default choice |
| Automated unit/integration tests | ✅ Ideal — no external infra |
| Single-process monolith (low traffic) | ✅ Acceptable |
| Multi-process / containerised production | ❌ Use `event-bus-redis` |
| Reliability/durability required | ❌ Use `event-bus-redis` |

## Related Modules

- [`event-bus-redis`](../event-bus-redis/README.md) — production-grade Redis Streams-based alternative.
- [`workflow-engine-inmemory`](../workflow-engine-inmemory/README.md) — typically paired with this module in dev.
