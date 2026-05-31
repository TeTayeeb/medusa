# Settings Module

## Overview

The Settings module provides a generic, schema-less key-value store for system-wide and scoped configuration. Unlike domain-specific configuration (e.g., store currencies), Settings is designed to persist arbitrary operational preferences that do not justify a dedicated data model.

It is used internally to store defaults such as the preferred region, default currency code, email sender configuration, and feature flags. External plugins and custom modules can write their own namespaced keys without schema migrations.

## Key Features

- **Generic KV storage**: Values are stored as flexible JSON blobs, supporting strings, numbers, booleans, and nested objects.
- **Scoped settings**: Each setting belongs to a scope — `global`, `store:{id}`, or `user:{id}` — allowing the same key to carry different values at different levels.
- **Scope resolution**: A setting lookup walks the scope hierarchy (user → store → global) and returns the most specific value found.
- **No fixed schema**: Consumers define their own key names; the module imposes no structure beyond scope and key.
- **Admin API**: The existing `/admin/stores` endpoint surfaces store-scoped settings for the dashboard.
- **Namespacing convention**: Keys should be prefixed with the owning module/plugin slug (e.g., `analytics.opt_out`, `email.from_address`).

## Scopes

| Scope            | Description                                    |
|------------------|------------------------------------------------|
| `global`         | Platform-wide default, applies to all stores   |
| `store:{id}`     | Overrides global for a specific store          |
| `user:{id}`      | Overrides store/global for a specific user     |

## API Usage

```typescript
const settingsService = container.resolve("settingsModuleService")

// Write
await settingsService.set("analytics.opt_out", true, { scope: "user:usr_01" })

// Read (with hierarchy resolution)
const value = await settingsService.get("analytics.opt_out", {
  scope: "user:usr_01",
  fallback: ["store:str_01", "global"],
})

// Delete
await settingsService.delete("analytics.opt_out", { scope: "user:usr_01" })
```

## Common Keys

| Key                        | Scope   | Default  | Description                    |
|----------------------------|---------|----------|--------------------------------|
| `store.default_currency`   | store   | `usd`    | ISO 4217 currency code         |
| `store.default_region`     | store   | —        | Default region ID              |
| `email.from_address`       | global  | —        | Outgoing email sender          |
| `analytics.opt_out`        | user    | `false`  | Analytics collection opt-out   |
| `checkout.guest_enabled`   | store   | `true`   | Allow guest checkout           |

## Admin API

Settings surfaced through `/admin/stores` include all `store:{id}` scoped keys for the active store. Updates via `PUT /admin/stores/:id` persist to the `store:{id}` scope.

## Module Registration

```typescript
{
  resolve: "@medusajs/settings",
}
```

## Dependencies

| Dependency           | Purpose                      |
|----------------------|------------------------------|
| `@medusajs/framework` | Module container, DB access  |
| Store module         | Store entity for scoping     |
| User module          | User entity for scoping      |
