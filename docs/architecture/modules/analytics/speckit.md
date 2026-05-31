# SpecKit — Analytics Module

## Module Identity

| Attribute      | Value                                     |
|----------------|-------------------------------------------|
| Module Name    | `@medusajs/analytics`                     |
| Version        | 2.15.4                                    |
| Module Key     | `analytics`                               |
| Type           | Infrastructure / Observability            |
| Database Tables| None (delegates to Settings module)       |
| Event Emitter  | No                                        |
| Event Consumer | No                                        |

---

## Functional Specifications

### SPEC-ANALYTICS-001: Provider Abstraction
**Description**: The module MUST expose a stable `IAnalyticsProvider` interface. All provider implementations MUST implement this interface without deviation.  
**Acceptance**: Swapping the active provider in config produces identical API behaviour from the consumer's perspective.

### SPEC-ANALYTICS-002: No-Op Local Provider
**Description**: The `local` provider MUST silently discard all calls without throwing errors or performing I/O.  
**Acceptance**: In test/dev mode, no network requests are made. `track()` returns resolved Promise immediately.

### SPEC-ANALYTICS-003: Opt-Out Enforcement
**Description**: When `analytics.opt_out` is `true` for the current user scope (or globally), ALL provider calls MUST be skipped before reaching the provider.  
**Acceptance**: PostHog receives zero events for an opted-out user. Test: mock provider; assert call count is 0 after opt-out set.

### SPEC-ANALYTICS-004: PII Exclusion
**Description**: No event property may contain email addresses, full names, phone numbers, or other direct identifiers. Anonymous IDs must be derived via one-way hash.  
**Acceptance**: Lint rule enforces no `email`, `name`, `phone` keys in analytics call sites.

### SPEC-ANALYTICS-005: Event Buffering (PostHog)
**Description**: The PostHog provider MUST buffer events in memory and flush either every 30 seconds or when the buffer reaches 20 events, whichever comes first.  
**Acceptance**: With 19 events, no flush occurs until 30s timer fires or 20th event arrives.

### SPEC-ANALYTICS-006: Error Isolation
**Description**: Any exception thrown by a provider MUST be caught and logged at `debug` level. The exception MUST NOT propagate to the calling code.  
**Acceptance**: Provider throws; calling function receives resolved (not rejected) promise.

### SPEC-ANALYTICS-007: Shutdown Flush
**Description**: On application shutdown (`onApplicationShutdown`), the PostHog provider MUST flush all buffered events before the process exits.  
**Acceptance**: After `shutdown()` called, PostHog SDK `shutdown()` is invoked; buffer is empty.

---

## Non-Functional Specifications

### SPEC-ANALYTICS-NFR-001: Performance
**Description**: `track()` call overhead MUST be < 1ms on the calling thread (excluding async I/O).  
**Target**: p99 < 1ms measured via micro-benchmark.

### SPEC-ANALYTICS-NFR-002: No Required Infrastructure
**Description**: The module MUST work in development with zero external dependencies (`local` provider default).  
**Target**: `medusa start` succeeds with no analytics config provided.

---

## API Contract

### `POST /admin/analytics/track`

| Field       | Type   | Required | Description              |
|-------------|--------|----------|--------------------------|
| `event`     | string | Yes      | Event name               |
| `properties`| object | No       | Scrubbed event properties|

**Response**: `200 OK` — `{ success: true }`

### `POST /admin/analytics/opt-out`

**Request body**: none  
**Effect**: Sets `analytics.opt_out = true` for `user:{actorId}` scope via Settings module.  
**Response**: `200 OK` — `{ opted_out: true }`

---

## Configuration Schema

```typescript
interface AnalyticsModuleOptions {
  provider?: "local" | "posthog" | string   // default: "local"
  posthog_api_key?: string                   // required if provider = "posthog"
  opt_out?: boolean                          // global opt-out default
  flush_interval?: number                    // ms; default 30000
  flush_at?: number                          // event count; default 20
}
```

---

## Test Checklist

- [ ] Local provider: `track()` resolves without network I/O
- [ ] PostHog provider: `track()` calls `posthog.capture()` with correct payload
- [ ] Opt-out set → no provider call made
- [ ] Provider throws → calling code receives resolved promise
- [ ] Shutdown flushes PostHog buffer
- [ ] PII lint rule blocks `email` property in analytics calls
- [ ] Anonymous ID changes across users (no shared hash)

---

## Dependencies & Interfaces

| Dependency           | Interface Used            | Direction |
|----------------------|---------------------------|-----------|
| Settings module      | `get("analytics.opt_out")` | Outbound |
| Auth middleware      | `req.auth_context`         | Inbound  |
| posthog-node         | `PostHog` SDK              | Outbound |
