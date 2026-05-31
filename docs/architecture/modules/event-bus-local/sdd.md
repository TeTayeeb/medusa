# Software Design Document — Event Bus Local

**Module:** `@medusajs/event-bus-local`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `event-bus-local` module implements Medusa's `IEventBusModuleService` contract using Node.js `EventEmitter`. It is the default event transport for single-process environments where operational simplicity is prioritised over durability.

---

## 2. Goals and Non-Goals

### Goals
- Provide a working, zero-config event bus for local development and testing.
- Satisfy the full `IEventBusModuleService` interface so modules can be developed against it.
- Support transactional event batching (event groups).

### Non-Goals
- Durable, persistent event delivery.
- Cross-process event propagation.
- Dead-letter queues or retry scheduling.

---

## 3. Architecture

### 3.1 Component Model

```
┌──────────────────────────────────────────┐
│           Medusa Container (DI)           │
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │     EventBusLocalService            │ │
│  │  ─────────────────────────────────  │ │
│  │  + emit(event, data)                │ │
│  │  + subscribe(event, handler)        │ │
│  │  + unsubscribe(event, handler)      │ │
│  │                                     │ │
│  │  ┌───────────────────────────────┐  │ │
│  │  │  Node.js EventEmitter         │  │ │
│  │  │  (internal, wrapped)          │  │ │
│  │  └───────────────────────────────┘  │ │
│  │                                     │ │
│  │  ┌───────────────────────────────┐  │ │
│  │  │  EventGroupingUtils           │  │ │
│  │  │  (transactional batching)     │  │ │
│  │  └───────────────────────────────┘  │ │
│  └─────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `EventBusLocalService` | Core service; wraps EventEmitter, exposes `IEventBusModuleService` |
| `EventBusUtils` (shared) | `groupEventsByQueue` — collects events during a transaction, flushes at commit |
| Module definition | Registers service in Medusa's DI container |

### 3.3 Subscription Registration

Subscribers are registered at application bootstrap by each Medusa module that needs to react to domain events. The registry is a `Map<eventName, Set<Subscriber>>` maintained by `EventBusLocalService`.

```typescript
// Subscriber registered by order module
eventBus.subscribe("order.placed", async ({ data }) => {
  await notificationService.sendOrderConfirmation(data.id)
})
```

### 3.4 Emit Flow

1. Caller invokes `eventBus.emit("order.placed", { id: "ord_01..." })`.
2. `EventBusLocalService` looks up all subscribers for `"order.placed"`.
3. Each subscriber is called sequentially (or in parallel, depending on config).
4. `emit()` resolves when all subscribers have returned.

### 3.5 Event Grouping (Transactional Batching)

When a Medusa service needs to emit multiple events atomically at the end of a database transaction, it uses event groups:

```typescript
// During service method
EventBusUtils.groupEventsByQueue([
  { eventName: "order.placed",    data: { id } },
  { eventName: "inventory.reserved", data: { orderId: id } },
], context.eventGroupId)

// At transaction commit, the engine flushes the group
await eventBus.emit(pendingGroupEvents)
```

---

## 4. Data Structures

```typescript
type Subscriber = (data: { eventName: string; data: unknown }) => Promise<void>

interface SubscriberContext {
  subscriberId: string
}

// Internal registry
private subscribers: Map<string, Map<string, Subscriber>>
// Map<eventName, Map<subscriberId, Subscriber>>
```

---

## 5. Error Handling

- If a subscriber throws, `EventBusLocalService` logs the error and continues invoking remaining subscribers for that event (fail-open per subscriber).
- No retry is attempted; the failure is recorded in application logs only.

---

## 6. Configuration Schema

```typescript
interface EventBusLocalOptions {
  // No configuration parameters required
}
```

---

## 7. Testing Strategy

- Unit tests mock the `EventEmitter` to assert subscription registration and emission.
- Integration tests verify end-to-end flow: emit → subscriber invoked → side effects visible.
- The module itself is used as the default in all Medusa integration test suites.

---

## 8. Dependencies

| Dependency | Purpose |
|---|---|
| `events` (Node.js built-in) | `EventEmitter` implementation |
| `@medusajs/framework/types` | `IEventBusModuleService` interface |
| `@medusajs/framework/utils` | `EventBusUtils`, module helpers |

---

## 9. Known Limitations & Future Work

- No persistence: considered by design for local use.
- A future enhancement could add optional fan-out to multiple in-process queues for complex test scenarios.
