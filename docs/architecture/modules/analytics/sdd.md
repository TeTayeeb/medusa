# Software Design Document — Analytics Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa Analytics module (v2.15.4). It covers the provider interface, event emission pipeline, data flow through the admin dashboard, opt-out mechanics, and integration points with other modules.

## 2. Architecture Overview

The Analytics module is structured in three layers:

1. **Module Service** (`AnalyticsModuleService`) — receives tracking calls and delegates to the active provider.
2. **Provider Interface** (`IAnalyticsProvider`) — a stable contract implemented by each backend adapter.
3. **Provider Implementations** — `LocalAnalyticsProvider` (no-op), `PostHogAnalyticsProvider`, and any custom provider.

```
Admin Dashboard
  ↓  useAnalytics() hook
AnalyticsModuleService.track(event, properties)
  ↓
IAnalyticsProvider → LocalAnalyticsProvider | PostHogAnalyticsProvider | CustomProvider
```

## 3. Data Flow

### 3.1 Page View Tracking

When the admin dashboard navigates to a new route, the React router integration fires `analytics.page(routeName)`. The hook resolves the module service from the Medusa container (via the admin API proxy) and calls `track("page_viewed", { route })`.

### 3.2 Feature Event Tracking

Individual admin UI components call `analytics.track(eventName, properties)` at key interaction points. Properties are scrubbed of PII before dispatch: user IDs are hashed, email addresses are excluded.

### 3.3 Opt-Out Check

Before delegating to the provider, `AnalyticsModuleService` reads the `analytics.opt_out` setting for the current session's user scope. If `true`, the method returns immediately without calling the provider.

## 4. Data Model

The Analytics module itself has **no database tables**. All persistent state (opt-out preference) is stored via the Settings module under the `analytics.*` key namespace.

## 5. Event Schema

```typescript
interface AnalyticsEvent {
  event: string           // e.g., "product_created"
  anonymousId: string     // SHA-256 of session token
  timestamp: string       // ISO 8601
  properties: {
    route?: string
    resource_type?: string
    resource_count?: number
    [key: string]: unknown // no PII allowed
  }
}
```

## 6. Provider Resolution

Providers are registered as named services in the Medusa IoC container:

```
analytics_provider_local
analytics_provider_posthog
analytics_provider_{custom_identifier}
```

The active provider is resolved at module boot time from `options.provider`. Default is `local`.

## 7. PostHog Provider Design

The PostHog provider wraps the `posthog-node` SDK. It:
- Initialises a `PostHog` client with the API key on `onApplicationBootstrap`.
- Buffers events in-memory and flushes every 30 seconds or when the buffer reaches 20 events.
- Calls `client.shutdown()` on application teardown to flush any remaining events.

## 8. Opt-Out Implementation

```typescript
async track(event: string, properties = {}) {
  const optOut = await this.settingsService.get("analytics.opt_out", {
    scope: `user:${this.currentUserId}`,
    fallback: ["global"],
  })
  if (optOut) return
  await this.provider.track(event, { ...properties, anonymousId: this.anonId })
}
```

## 9. Admin Dashboard Integration

The dashboard exposes a `useAnalytics()` React hook that:
1. Reads the current user's anonymous ID from a session-stable cookie.
2. Resolves opt-out status from the current user preferences.
3. Exposes `track()` and `page()` methods to components.

## 10. Error Handling

Analytics failures must never break the user-facing request. All provider calls are wrapped:

```typescript
try {
  await this.provider.track(event, properties)
} catch {
  // log at debug level; swallow silently
}
```

## 11. Security Considerations

- No PII in event properties. Enforced by linting rules on the analytics call sites.
- Anonymous IDs are one-way hashes; they cannot be reversed to user identities.
- API keys for third-party providers are stored only in environment variables, never in the database.

## 12. Configuration Options

| Option            | Type    | Default   | Description                          |
|-------------------|---------|-----------|--------------------------------------|
| `provider`        | string  | `local`   | Active provider identifier           |
| `posthog_api_key` | string  | —         | PostHog project API key              |
| `opt_out`         | boolean | `false`   | Global opt-out override              |
| `flush_interval`  | number  | `30000`   | PostHog flush interval (ms)          |
| `flush_at`        | number  | `20`      | PostHog flush event count threshold  |
