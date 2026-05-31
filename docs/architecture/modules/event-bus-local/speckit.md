# SpecKit — Event Bus Local

**Module:** `@medusajs/event-bus-local`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/event-bus-local` |
| DI key | `Modules.EVENT_BUS` |
| Interface | `IEventBusModuleService` |
| Category | Infrastructure / Event Transport |
| Default for | `NODE_ENV=development` |

---

## 2. Functional Specifications

### SPEC-EBL-001 — Subscribe
**Given** a Medusa module calls `eventBus.subscribe("order.placed", handler, { subscriberId: "s1" })`
**Then** `handler` is stored in the subscriber registry under key `"order.placed"` with id `"s1"`.
**And** subsequent `emit("order.placed", ...)` calls invoke `handler`.

### SPEC-EBL-002 — Unsubscribe
**Given** `handler` is subscribed to `"order.placed"` with `subscriberId: "s1"`
**When** `eventBus.unsubscribe("order.placed", handler, { subscriberId: "s1" })` is called
**Then** `handler` is removed from the registry.
**And** subsequent `emit("order.placed", ...)` calls do NOT invoke `handler`.

### SPEC-EBL-003 — Emit single event
**Given** handlers `h1`, `h2` are subscribed to `"order.placed"`
**When** `eventBus.emit("order.placed", { id: "ord_01" })` is called
**Then** both `h1` and `h2` are invoked with `{ eventName: "order.placed", data: { id: "ord_01" } }`.
**And** `emit()` resolves only after all handlers have settled.

### SPEC-EBL-004 — Emit array of events
**Given** multiple events are batched: `[{ eventName: "A", data: d1 }, { eventName: "B", data: d2 }]`
**When** `eventBus.emit(events)` is called
**Then** each event is dispatched to its respective subscribers.
**And** order of dispatch matches the array order.

### SPEC-EBL-005 — Subscriber error isolation
**Given** `h1` throws an error and `h2` is a subsequent subscriber to the same event
**When** the event is emitted
**Then** `h2` is still invoked.
**And** the error from `h1` is logged but does not propagate to the emitter.
**And** `emit()` resolves (does not reject) despite `h1`'s failure.

### SPEC-EBL-006 — Event grouping
**Given** events are accumulated into an event group with `groupId: "g1"` during a transaction
**When** the transaction commits and `emit(groupedEvents)` is called
**Then** all grouped events are dispatched atomically in insertion order.

### SPEC-EBL-007 — No-op emit
**Given** no subscribers exist for `"unknown.event"`
**When** `eventBus.emit("unknown.event", {})` is called
**Then** `emit()` resolves immediately with no side effects and no error.

---

## 3. Non-Functional Specifications

### SPEC-EBL-NF-001 — Zero external dependencies
The module must have no runtime npm dependencies outside `@medusajs/framework`.

### SPEC-EBL-NF-002 — No external I/O
All operations must be in-process. No network calls, no file system access.

### SPEC-EBL-NF-003 — Interface compatibility
The module must satisfy `IEventBusModuleService` exactly as defined in `@medusajs/framework/types`. No additional public methods may be required by consumers.

---

## 4. Configuration Specification

```typescript
// No required options. Module registers with empty options object.
interface EventBusLocalOptions {}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-EBL-001 | Unit | Required |
| SPEC-EBL-002 | Unit | Required |
| SPEC-EBL-003 | Unit | Required |
| SPEC-EBL-004 | Unit | Required |
| SPEC-EBL-005 | Unit | Required |
| SPEC-EBL-006 | Integration | Required |
| SPEC-EBL-007 | Unit | Required |
| SPEC-EBL-NF-001 | Static analysis | Required |
| SPEC-EBL-NF-002 | Unit (no mocks needed) | Required |
| SPEC-EBL-NF-003 | TypeScript compilation | Required |

---

## 6. Out of Scope

- Persistent event storage
- Cross-process event delivery
- Retry scheduling
- Dead-letter queues
- Message ordering guarantees across concurrent emits
