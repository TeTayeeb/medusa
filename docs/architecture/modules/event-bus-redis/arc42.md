# arc42 Architecture Documentation — Event Bus Redis

**Module:** `@medusajs/event-bus-redis`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The event-bus-redis module must provide durable, distributed event delivery across multiple Medusa processes. It must support consumer-group-based load balancing, survive process restarts, and maintain the `IEventBusModuleService` interface so the rest of Medusa is unaware of the transport.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Durability** | Events survive API process restart before they are consumed |
| 2 | **Scalability** | Multiple workers consume events concurrently without duplicates |
| 3 | **Interface parity** | Drop-in replacement for `event-bus-local` |

---

## 2. Architecture Constraints

- Must use Redis ≥ 5.0 (Redis Streams API).
- Must not change the `IEventBusModuleService` interface.
- Consumer group names and stream keys must be configurable to support multi-tenant deployments.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────────────┐
│ External: Redis Server (≥ 5.0)                              │
│   Streams: medusa:order.placed, medusa:product.updated ...  │
│   Consumer Group: medusa-consumer-group                     │
└────────────────────┬───────────────────────────────────────┘
                     │ ioredis
        ┌────────────┴─────────────┐
        │  API Process             │  Worker Process
        │  emit() → XADD           │  StreamPoller → XREADGROUP → handlers
        └──────────────────────────┘
```

---

## 4. Solution Strategy

Emit maps to `XADD` (append to stream). Consumption maps to `XREADGROUP` inside a polling loop (`StreamPoller`). Acknowledgements (`XACK`) are sent on successful handler completion. Failed messages remain in the Pending Entries List and are re-delivered after a configurable claim timeout.

---

## 5. Building Blocks

### Level 1: Module
```
event-bus-redis
  ├── EventBusRedisService    (IEventBusModuleService impl)
  ├── StreamPoller            (background polling loop)
  └── RedisConnection         (ioredis lifecycle)
```

### Level 2: EventBusRedisService

| Method | Redis operation |
|---|---|
| `emit(event, data)` | `XADD medusa:<event> * fields...` |
| `subscribe(event, fn)` | Registers fn; creates consumer group if needed |
| `unsubscribe(event, fn)` | Removes fn; stops polling if no subscribers remain |

---

## 6. Runtime View

### Scenario: Distributed Order Event

```
API Process (emit)               Redis Stream                  Worker (consume)
     │                                │                              │
     │── XADD medusa:order.placed ──► │ ◄─ entry id1 stored         │
     │                                │                              │
     │                                │ ◄── XREADGROUP ─────────────│
     │                                │ ──► [id1, data] ────────────►│
     │                                │                  handler()   │
     │                                │ ◄── XACK id1 ───────────────│
```

---

## 7. Deployment View

```
┌──────── Kubernetes Cluster ───────────────────────────────────┐
│                                                               │
│  ┌─────────────────┐    ┌──────────────┐    ┌─────────────┐  │
│  │  API Pod ×3      │    │ Worker Pod×2 │    │  Redis Pod  │  │
│  │  emit() only     │    │  XREADGROUP  │    │  (Streams)  │  │
│  └─────────────────┘    └──────────────┘    └─────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### At-Least-Once Delivery
Redis Streams + consumer groups guarantee at-least-once delivery. Subscribers must be idempotent.

### Message Ordering
Messages within a single stream are ordered. Across streams, no ordering guarantee exists.

### Back-pressure
If consumers lag, the stream grows. Redis memory limits should be monitored; `MAXLEN` can be set on streams to cap growth.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Redis Streams over Pub/Sub | Streams provide persistence and consumer groups; Pub/Sub is fire-and-forget |
| AD-2 | `ioredis` client | Battle-tested, cluster-aware, TypeScript-typed |
| AD-3 | Polling (XREADGROUP BLOCK) | Avoids busy-waiting; Redis BLOCK suspends until new messages arrive |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Durability | Events survive API restart; verified by integration test |
| Load balancing | 2-worker test shows each event processed exactly once |
| Config flexibility | `streamName`, `consumerGroupName`, `batchSize` all configurable |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Redis unavailability | High | Use Redis Sentinel or Cluster; implement circuit breaker |
| PEL growth on worker crash | Medium | Monitor PEL size; implement `XAUTOCLAIM` re-delivery |
| No built-in DLQ | Medium | Implement application-level DLQ for poison messages |
