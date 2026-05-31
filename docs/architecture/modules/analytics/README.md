# Analytics Module

## Overview

The Analytics module provides an abstraction layer for tracking behavioral events within the Medusa platform. It decouples event instrumentation from specific third-party analytics providers, enabling teams to switch or combine analytics backends without modifying application code.

The module is primarily consumed by the admin dashboard to capture UX telemetry such as page views, feature interactions, and API call patterns. It is designed with privacy as a first-class concern: no personally identifiable information (PII) is included in any tracked event.

## Key Features

- **Provider abstraction**: Defines a uniform `IAnalyticsProvider` interface implemented by pluggable backends.
- **Built-in providers**: Ships with a `local` (no-op) provider for development and a `PostHog` provider for production telemetry.
- **Custom providers**: Any provider implementing the interface can be registered via Medusa's module configuration.
- **Event types tracked**: page views, API endpoint calls, admin feature usage (e.g., discount creation, product publish).
- **Privacy-respecting**: Event payloads contain only anonymised identifiers (session IDs, not user emails or names).
- **Opt-out support**: Analytics collection can be disabled globally or per-session via configuration or a user-level preference flag.

## Module Registration

```typescript
// medusa-config.ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/analytics",
      options: {
        provider: "posthog",
        posthog_api_key: process.env.POSTHOG_API_KEY,
        opt_out: false,
      },
    },
  ],
})
```

## Provider Interface

```typescript
interface IAnalyticsProvider {
  track(event: string, properties?: Record<string, unknown>): Promise<void>
  page(name: string, properties?: Record<string, unknown>): Promise<void>
  identify(anonymousId: string, traits?: Record<string, unknown>): Promise<void>
  optOut(): Promise<void>
}
```

## Event Categories

| Category      | Example Events                                    |
|---------------|---------------------------------------------------|
| Navigation    | `page_viewed`, `tab_changed`                      |
| Products      | `product_created`, `product_published`            |
| Orders        | `order_list_viewed`, `order_export_triggered`     |
| Discounts     | `discount_rule_created`, `promotion_applied`      |
| Settings      | `store_settings_updated`, `region_created`        |
| Auth          | `admin_logged_in`, `session_expired`              |

## Privacy & Opt-Out

Events are associated with an ephemeral anonymous session ID generated at login. No email addresses, names, or other PII appear in event properties. The opt-out flag can be set:

- **Globally** via `options.opt_out: true` in `medusa-config.ts`.
- **Per-session** via the `POST /admin/analytics/opt-out` endpoint.

When opted out, all provider calls are silently swallowed by the no-op local provider.

## Dependencies

| Dependency           | Purpose                              |
|----------------------|--------------------------------------|
| `@medusajs/framework` | Module container, event bus types    |
| Admin dashboard      | Primary consumer of the provider API |

## Related Modules

- **Settings** – stores the opt-out preference per store.
- **Auth** – session identity used as anonymous ID seed.
