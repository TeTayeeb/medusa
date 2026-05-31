# Software Design Document — Event Bus Redis

**Module:** `@medusajs/event-bus-redis`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `event-bus-redis` module implements `IEventBusModuleService` using Redis Streams. It supports durable, distributed event delivery with consumer-group-based load balancing across multiple Medusa worker processes.

---

## 2. Goals and Non-Goals

### Goals
- Deliver domain events reliably across multiple processes.
- Support horizontal scaling of Medusa workers.
- Retain events in Redis until they are successfully acknowledged.
- Maintain the same `IEventBusModuleService` interface as the local variant.

### Non-Goals
- Cross-datacenter replication.
- Built-in dead-letter queue (DLQ) management.
- Event replay / rewind beyond the Redis stream retention window.

---

## 3. Architecture

### 3.1 Component Model

```
┌──────────────── API Process ──────────────────┐
│  EventBusRedisService                          │
│    emit("order.placed", data)                  │
│      └──► XADD medusa:order.placed * data ... │
└────────────────────────────────────────────────┘
                     │
               Redis Stream
               medusa:order.placed
                     │
┌──────────── Worker Process ───────────────────┐
│  StreamPoller                                  │
│    XREADGROUP GROUP medusa-workers C1 ...      │
│      └──► EventBusRedisService.handleMessage() │
│             └──► subscriber(event)             │
│               └──► XACK on success            │
└────────────────────────────────────────────────┘
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `EventBusRedisService` | Core service; `emit()`, `subscribe()`, `unsubscribe()` |
| `StreamPoller` | Background loop using `XREADGROUP` to pull messages from all subscribed streams |
| `RedisConnection` | Manages `ioredis` client lifecycle; handles reconnection |
| Module definition | Registers service + starts poller in DI container |

### 3.3 Stream Naming Convention

Each event name maps to a Redis stream key:

```
medusa:<eventName>
// e.g. medusa:order.placed, medusa:product.updated
```

### 3.4 Emit Flow

1. `emit("order.placed", payload)` serialises `{ eventName, data }` to JSON.
2. `XADD medusa:order.placed * eventName order.placed data <json>` appends to stream.
3. Redis assigns a stream entry ID (`<timestamp>-<seq>`).
4. `emit()` resolves immediately after the XADD completes.

### 3.5 Consume Flow

1. `StreamPoller` runs `XREADGROUP GROUP medusa-workers <consumerId> COUNT <batchSize> BLOCK <pollInterval> STREAMS medusa:order.placed >` per subscribed stream.
2. For each received message, it invokes the registered `Subscriber`.
3. On success: `XACK medusa:order.placed <messageId>`.
4. On failure: message remains in the Pending Entries List (PEL); a future `XAUTOCLAIM` will re-deliver it.

### 3.6 Consumer Group Management

Consumer groups are created with `XGROUP CREATE medusa:order.placed medusa-workers 0 MKSTREAM` on first subscription. If the group already exists, the `BUSYGROUP` error is swallowed.

---

## 4. Data Structures

```typescript
interface RedisStreamMessage {
  eventName: string
  data: string          // JSON-serialised event payload
  options?: string      // JSON-serialised emit options
}

interface EventBusRedisOptions {
  redisUrl: string
  streamName?: string             // default: "medusa-event-bus"
  consumerGroupName?: string      // default: "medusa-consumer-group"
  batchSize?: number              // default: 100
  pollInterval?: number           // default: 3000 (ms)
  redisOptions?: RedisOptions     // ioredis options passthrough
}
```

---

## 5. Error Handling

- **Subscriber failure** — message stays in PEL; after `XCLAIM` timeout it is re-delivered.
- **Redis connection loss** — `ioredis` retries with exponential backoff; poller resumes automatically.
- **Serialisation failure** — message is acked and error is logged (bad data should not block the stream).

---

## 6. Concurrency and Ordering

- Multiple workers in the same consumer group receive disjoint subsets of messages (competing consumers); this provides load balancing but not strict ordering across workers.
- Within a single worker, messages from the same stream are processed sequentially.

---

## 7. Dependencies

| Dependency | Purpose |
|---|---|
| `ioredis` | Redis client (XADD, XREADGROUP, XACK) |
| `@medusajs/framework/types` | `IEventBusModuleService` interface |
| `@medusajs/framework/utils` | `EventBusUtils`, module helpers |

---

## 8. Testing Strategy

- Unit tests mock `ioredis` and assert correct XADD / XREADGROUP calls.
- Integration tests spin up a real Redis instance via Docker and verify full emit→consume→ack cycle.
- Consumer group creation idempotency is validated in integration tests.
