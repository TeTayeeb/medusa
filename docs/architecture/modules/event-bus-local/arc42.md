# arc42 Architecture Documentation — Event Bus Local

**Module:** `@medusajs/event-bus-local`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The event-bus-local module must provide a fully functional publish/subscribe mechanism within a single Node.js process. It exists to satisfy Medusa's `IEventBusModuleService` contract with zero external infrastructure, enabling developers to run and test Medusa without Redis or any message broker.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Simplicity** | Zero config, zero dependencies — `npm install` and go |
| 2 | **Compatibility** | 100% interface-compatible with `event-bus-redis` |
| 3 | **Testability** | Tests run in-process without Docker or external services |

---

## 2. Architecture Constraints

- Must not introduce any npm dependencies beyond what is already in `@medusajs/framework`.
- Must implement `IEventBusModuleService` exactly — no extensions that break the substitution principle.
- Must support transactional event grouping as defined by `EventBusUtils`.

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────────┐
│                    Medusa Application                       │
│                                                            │
│   Order Module ─────► eventBus.emit("order.placed")        │
│   Notification Module ◄── eventBus.subscribe(handler)      │
│   Inventory Module   ◄── eventBus.subscribe(handler)       │
│                                                            │
│         ┌───────────────────────────────────┐             │
│         │     event-bus-local               │             │
│         │  (all within the same process)    │             │
│         └───────────────────────────────────┘             │
└────────────────────────────────────────────────────────────┘
```

**External interfaces:** None. All communication is in-process via Node.js `EventEmitter`.

---

## 4. Solution Strategy

The module wraps Node.js `EventEmitter` behind the `IEventBusModuleService` facade. A subscriber registry (`Map<eventName, Map<subscriberId, fn>>`) is maintained by the service. Event grouping is handled by `EventBusUtils.groupEventsByQueue`, which accumulates events during a database transaction and flushes them atomically at commit time.

---

## 5. Building Blocks

### Level 1: Module
```
event-bus-local
  └── EventBusLocalService    (IEventBusModuleService impl)
        └── EventEmitter      (Node.js built-in)
        └── SubscriberRegistry (Map)
```

### Level 2: EventBusLocalService

| Method | Behaviour |
|---|---|
| `subscribe(event, fn)` | Adds `fn` to `SubscriberRegistry[event]` |
| `unsubscribe(event, fn)` | Removes `fn` from registry |
| `emit(event, data)` | Iterates registry, calls each subscriber |

---

## 6. Runtime View

### Scenario: Order Placed

```
HTTP Request
  │
  ▼
OrderService.createOrder()
  ├── DB: INSERT order
  ├── EventBusUtils.groupEvents(["order.placed"], context.eventGroupId)
  └── DB COMMIT
        │
        └──► eventBus.emit([{ eventName: "order.placed", data: { id } }])
               │
               ├──► NotificationSubscriber({ data }) → sends email
               └──► InventorySubscriber({ data }) → reserves stock
```

---

## 7. Deployment View

The local event bus has no deployment artifact of its own. It runs embedded within the Medusa Node.js process. No infrastructure provisioning is required.

```
Single Node.js Process
┌──────────────────────────────┐
│  Medusa Server               │
│  + EventBusLocalService      │
│  + All Subscribers           │
└──────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### Error Handling
Subscriber failures are logged and do not propagate to the emitter. This ensures a failing notification handler does not roll back an order creation.

### Logging
All emit operations are logged at `debug` level with event name and truncated payload.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Use `EventEmitter`, not a queue | Simplest, zero-dependency approach for single-process use |
| AD-2 | Fail-open on subscriber errors | Preserve primary operation integrity even if a side-effect subscriber fails |
| AD-3 | No retry | Retries imply state and scheduling; out of scope for local dev |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Zero config startup | Module registers with no required options |
| Interface parity | All 4 `IEventBusModuleService` methods implemented |
| No external I/O | No network calls; verified by unit tests with no mocks needed |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Memory leak via unbounded subscriptions | Medium | Callers must `unsubscribe` at teardown |
| Lost events on crash | Low (dev only) | By design; use `event-bus-redis` in production |
