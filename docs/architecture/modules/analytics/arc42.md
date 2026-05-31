# arc42 Architecture Document — Analytics Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The Analytics module must enable behavioural telemetry of admin dashboard usage without exposing PII, support swappable provider backends, and respect user opt-out preferences — all without impacting the performance or correctness of the primary application.

### 1.2 Quality Goals

| Priority | Quality Goal       | Scenario                                                   |
|----------|--------------------|------------------------------------------------------------|
| 1        | Privacy            | No user email or name ever appears in any event payload    |
| 2        | Resilience         | Provider failure never propagates to the user request      |
| 3        | Replaceability     | Switch from PostHog to a custom provider without code changes to consumers |
| 4        | Performance        | `track()` call adds < 1ms overhead to the calling thread   |

### 1.3 Stakeholders

| Role              | Expectation                                            |
|-------------------|--------------------------------------------------------|
| Platform engineer | Simple provider swap via config; no SDK fragmentation  |
| Compliance officer| GDPR-compliant: opt-out persisted, no PII at rest      |
| Product team      | Reliable event delivery for UX decision making         |

---

## 2. Architecture Constraints

- Must integrate with Medusa's IoC container; no singleton global state.
- Must not introduce required infrastructure (e.g., no mandatory Redis).
- Event payloads must pass a PII lint check before merging.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────┐
│           Medusa Admin Dashboard        │
│  useAnalytics() hook                    │
│    track() / page() calls               │
└────────────────┬────────────────────────┘
                 │ HTTP (admin API proxy)
┌────────────────▼────────────────────────┐
│       AnalyticsModuleService            │
│  - opt-out check (Settings module)      │
│  - delegate to IAnalyticsProvider       │
└──────┬──────────────────────────────────┘
       │
  ┌────▼──────────────┐  ┌─────────────────────┐
  │  LocalProvider     │  │  PostHogProvider     │
  │  (no-op)          │  │  posthog-node SDK    │
  └───────────────────┘  └──────────────────────┘
```

External system: PostHog SaaS (events sent over HTTPS to `app.posthog.com`).

---

## 4. Solution Strategy

- **Provider pattern** isolates the module from third-party SDKs.
- **Swallow-all error handling** ensures analytics never breaks the application.
- **Settings module delegation** for opt-out avoids duplicating user preference storage.
- **Anonymous IDs** derived from session tokens (hashed) prevent identity reconstruction.

---

## 5. Building Block View

### Level 1

```
AnalyticsModule
  ├── AnalyticsModuleService   (orchestrator)
  ├── IAnalyticsProvider       (interface)
  ├── LocalAnalyticsProvider   (no-op implementation)
  └── PostHogAnalyticsProvider (PostHog implementation)
```

### Level 2 — PostHogAnalyticsProvider

```
PostHogAnalyticsProvider
  ├── PostHog client (posthog-node)
  ├── Event buffer (in-memory queue)
  ├── Flush timer (30s interval)
  └── Shutdown hook (flush remaining)
```

---

## 6. Runtime View

### Scenario: Page View Tracked

1. Admin navigates to `/orders`.
2. React router calls `analytics.page("order-list")`.
3. `useAnalytics` hook calls `POST /admin/analytics/track` with `{ event: "page_viewed", properties: { route: "/orders" } }`.
4. API route calls `AnalyticsModuleService.track()`.
5. Opt-out checked via Settings module.
6. `PostHogProvider.track()` called; event buffered.
7. Buffer flushed to PostHog every 30s.

---

## 7. Deployment View

The Analytics module runs in-process within the Medusa server. The PostHog provider sends events over HTTPS to PostHog's cloud API. No additional infrastructure is required unless self-hosting PostHog.

---

## 8. Cross-Cutting Concerns

### Error Handling
All provider calls are wrapped in `try/catch`. Errors are logged at `debug` level only.

### Opt-Out
The opt-out flag is persisted in the Settings module. It is checked on every `track()` call without caching (to respect real-time changes).

### Security
PostHog API keys are environment-variable-only. They are never stored in the database or logged.

---

## 9. Architecture Decisions

| ID  | Decision                                    | Rationale                                               |
|-----|---------------------------------------------|---------------------------------------------------------|
| AD1 | Provider pattern with IoC container         | Enables runtime swap without consumer code changes      |
| AD2 | Swallow analytics errors silently           | Analytics must never degrade the primary user flow      |
| AD3 | Delegate opt-out to Settings module         | Single source of truth for user preferences             |
| AD4 | Anonymous ID = hash of session token        | Enables session-level analytics without PII             |

---

## 10. Quality Scenarios

| Quality  | Scenario                                          | Measure                           |
|----------|---------------------------------------------------|-----------------------------------|
| Privacy  | User opts out → no events sent                    | Zero events in PostHog for user   |
| Perf     | 1000 concurrent admin users tracking events       | < 5ms p99 added latency           |
| Replace  | Switch provider in config → restart               | New provider active, no code changes |
