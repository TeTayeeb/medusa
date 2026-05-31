# Event Bus Redis Module

## Overview

The `event-bus-redis` module is Medusa's production-grade event bus implementation. It uses **Redis Streams** to provide durable, distributed, ordered event delivery across multiple Medusa worker and API processes. It implements the same `IEventBusModuleService` interface as `event-bus-local`, allowing transparent substitution based on environment.

## Purpose

In multi-process deployments (separate API servers and background workers), events must cross process boundaries reliably. Redis Streams natively support consumer groups, acknowledgements, and persistent storage of messages, making them an ideal transport for Medusa's domain events.

## Key Features

- **Durable delivery** — messages are persisted in Redis Streams until explicitly acknowledged (XACK).
- **Consumer groups** — multiple worker processes share a consumer group; Redis ensures each event is delivered to exactly one consumer per group (competing consumers / load balancing).
- **Ordered delivery** — within a stream, events are delivered in insertion order.
- **Configurable batching** — the `batchSize` option controls how many messages are read per `XREADGROUP` call.
- **Auto-creation of streams** — streams and consumer groups are created on first use.
- **Same interface** — satisfies `IEventBusModuleService`; the rest of Medusa has no awareness of the transport.

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

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/event-bus-redis",
      key: Modules.EVENT_BUS,
      options: {
        redisUrl: process.env.REDIS_URL,           // required
        streamName: "medusa-event-bus",            // default stream prefix
        consumerGroupName: "medusa-consumer-group",// default consumer group
        batchSize: 100,                             // messages per poll
        pollInterval: 3000,                         // ms between polls
      },
    },
  ],
})
```

## Redis Streams Architecture

```
Producer (Medusa API)          Redis Stream                  Consumer (Worker)
──────────────────────         ────────────────────          ────────────────────
emit("order.placed", {...}) ──► XADD medusa-event-bus ──►  XREADGROUP GROUP workers
                                  [id1] { eventName, data }  │  handler(event)
                                  [id2] { eventName, data }  └► XACK medusa-event-bus id1
```

## Limitations

| Constraint | Detail |
|---|---|
| **Redis required** | Needs a running Redis ≥ 5.0 instance. |
| **Message ordering per stream** | Cross-stream ordering is not guaranteed. |
| **No built-in DLQ** | Failed messages are not automatically moved to a dead-letter stream (requires custom handling). |

## When to Use

| Scenario | Recommendation |
|---|---|
| Production multi-process deployment | ✅ Required |
| Horizontal scaling of workers | ✅ Ideal |
| Durable event delivery needed | ✅ Use this |
| Local development / tests | ❌ Use `event-bus-local` |

## Related Modules

- [`event-bus-local`](../event-bus-local/README.md) — development alternative.
- [`workflow-engine-redis`](../workflow-engine-redis/README.md) — typically paired with this in production.
