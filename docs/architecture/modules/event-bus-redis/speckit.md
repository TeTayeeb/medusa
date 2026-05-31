# SpecKit — Event Bus Redis

**Module:** `@medusajs/event-bus-redis`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/event-bus-redis` |
| DI key | `Modules.EVENT_BUS` |
| Interface | `IEventBusModuleService` |
| Category | Infrastructure / Event Transport |
| Default for | Production multi-process deployments |
| External dependency | Redis ≥ 5.0 |

---

## 2. Functional Specifications

### SPEC-EBR-001 — Emit appends to Redis Stream
**Given** the module is configured with `redisUrl` pointing to a running Redis
**When** `eventBus.emit("order.placed", { id: "ord_01" })` is called
**Then** a new entry is appended to the Redis stream `medusa:order.placed` (or the configured stream prefix).
**And** the entry contains fields `eventName` and `data` (JSON-serialised payload).

### SPEC-EBR-002 — Consumer group creation on subscribe
**Given** no consumer group exists for `"order.placed"`
**When** `eventBus.subscribe("order.placed", handler)` is called for the first time
**Then** the consumer group is created: `XGROUP CREATE medusa:order.placed <groupName> 0 MKSTREAM`.
**And** subsequent subscribe calls with the same event name do NOT error (BUSYGROUP is swallowed).

### SPEC-EBR-003 — Consumer delivers to subscriber
**Given** `handler` is subscribed to `"order.placed"` and an event has been emitted
**When** the stream poller reads the message via `XREADGROUP`
**Then** `handler` is invoked with the deserialised event payload.
**And** `XACK` is sent for the message ID after successful handler completion.

### SPEC-EBR-004 — Failed message remains in PEL
**Given** `handler` throws an unhandled exception for a message
**When** the exception propagates
**Then** `XACK` is NOT sent.
**And** the message remains in the Pending Entries List (PEL) for re-delivery.

### SPEC-EBR-005 — Multiple consumers load-balance
**Given** two worker processes are subscribed to `"order.placed"` in the same consumer group
**When** 10 events are emitted
**Then** each event is delivered to exactly one of the two workers (no duplicates).

### SPEC-EBR-006 — Unsubscribe stops delivery
**When** `eventBus.unsubscribe("order.placed", handler)` is called
**Then** subsequent messages on the stream are not delivered to `handler`.

### SPEC-EBR-007 — Configurable stream/group names
**Given** `options.streamName = "my-stream"` and `options.consumerGroupName = "my-group"`
**Then** all XADD, XREADGROUP, and XACK calls use those names instead of defaults.

### SPEC-EBR-008 — Interface compatibility
The module must satisfy `IEventBusModuleService` exactly. All tests written against `event-bus-local` must pass unchanged against `event-bus-redis`.

---

## 3. Non-Functional Specifications

### SPEC-EBR-NF-001 — Durability
Messages emitted before a consumer is online must be deliverable after the consumer starts (stream retention).

### SPEC-EBR-NF-002 — Configurable batch size
The `batchSize` option must control the `COUNT` parameter of `XREADGROUP`.

### SPEC-EBR-NF-003 — Reconnection
The module must automatically reconnect to Redis on connection loss without requiring a process restart.

---

## 4. Configuration Specification

```typescript
interface EventBusRedisOptions {
  redisUrl: string                  // required
  streamName?: string               // default: "medusa-event-bus"
  consumerGroupName?: string        // default: "medusa-consumer-group"
  batchSize?: number                // default: 100
  pollInterval?: number             // default: 3000 (ms)
  redisOptions?: RedisOptions       // ioredis passthrough
}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-EBR-001 | Integration (real Redis) | Required |
| SPEC-EBR-002 | Integration | Required |
| SPEC-EBR-003 | Integration | Required |
| SPEC-EBR-004 | Integration | Required |
| SPEC-EBR-005 | Integration (2-process) | Required |
| SPEC-EBR-006 | Integration | Required |
| SPEC-EBR-007 | Unit | Required |
| SPEC-EBR-008 | TypeScript / contract tests | Required |
| SPEC-EBR-NF-001 | Integration | Required |
| SPEC-EBR-NF-002 | Unit | Required |
| SPEC-EBR-NF-003 | Integration | Required |

---

## 6. Out of Scope

- Dead-letter queue management
- Cross-datacenter replication
- Event replay beyond stream retention
- Strict per-key ordering across multiple workers
