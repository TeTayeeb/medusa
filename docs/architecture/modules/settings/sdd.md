# Software Design Document — Settings Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa Settings module (v2.15.4). It covers the storage model, scope resolution algorithm, API layer, and integration patterns for both core Medusa modules and third-party plugins.

## 2. Architecture Overview

The Settings module is intentionally minimal. It consists of:

1. **Data layer** — a single `setting` table with scope, key, and JSON value columns.
2. **Service layer** — `SettingsModuleService` with `get`, `set`, `delete`, and `list` operations.
3. **Scope resolver** — a hierarchy walker that resolves the most specific value for a key.
4. **Admin API bridge** — store-scoped settings are surfaced and mutated through the Store module's `/admin/stores` endpoint.

```
Consumer (module / plugin)
  ↓
SettingsModuleService.get(key, { scope, fallback })
  ↓
Scope Resolver → DB query per scope level (user → store → global)
  ↓
Returns first non-null value found, or undefined
```

## 3. Data Model

### Setting Entity

| Field        | Type      | Description                                        |
|--------------|-----------|----------------------------------------------------|
| `id`         | string    | Unique setting record ID (`set_*`)                 |
| `scope`      | string    | Scope identifier: `global`, `store:{id}`, `user:{id}` |
| `key`        | string    | Dot-namespaced key (e.g., `analytics.opt_out`)     |
| `value`      | JSONB     | Arbitrary JSON value                               |
| `created_at` | timestamp | Creation timestamp                                 |
| `updated_at` | timestamp | Last update timestamp                              |

**Unique constraint**: `(scope, key)` — one value per key per scope level.

**Index**: `(key, scope)` — fast lookup by key across scopes.

## 4. Scope Resolution Algorithm

```typescript
async get(key: string, options: { scope: string; fallback?: string[] }) {
  const scopeChain = [options.scope, ...(options.fallback ?? [])]

  for (const scope of scopeChain) {
    const setting = await this.settingRepository.findOne({ scope, key })
    if (setting !== null) return setting.value
  }

  return undefined
}
```

Scopes are evaluated left-to-right. The first non-null value wins. This enables clean override hierarchies:

```
user:usr_01 → store:str_01 → global
```

If none of the scopes have a value, `undefined` is returned. Callers are responsible for applying their own fallback defaults.

## 5. Write Operations

```typescript
async set(key: string, value: unknown, options: { scope: string }) {
  const existing = await this.settingRepository.findOne({
    scope: options.scope, key
  })

  if (existing) {
    return await this.settingRepository.update(existing.id, { value })
  }
  return await this.settingRepository.create({ scope: options.scope, key, value })
}
```

Upsert semantics: creates a new record or updates the existing one at the specified scope level without touching other scopes.

## 6. Key Namespacing Convention

All keys should follow the pattern `{owner_slug}.{key_name}`:

| Owner       | Example Key                    | Value Type |
|-------------|--------------------------------|------------|
| `store`     | `store.default_currency`       | string     |
| `store`     | `store.default_region`         | string     |
| `analytics` | `analytics.opt_out`            | boolean    |
| `email`     | `email.from_address`           | string     |
| `checkout`  | `checkout.guest_enabled`       | boolean    |
| `plugin.*`  | `acme_plugin.feature_x`        | any        |

Keys without a namespace prefix are reserved for core Medusa use.

## 7. Admin API Integration

Store-level settings are surfaced through the Store module's `/admin/stores/:id` endpoint. When a `PUT /admin/stores/:id` request arrives:

1. The Store module handler updates scalar fields on the `store` entity.
2. Any keys matching the `settings.*` prefix in the request body are forwarded to `SettingsModuleService.set()` with scope `store:{id}`.

This design keeps the Settings module hidden from the API consumer; settings feel like native store attributes.

## 8. Bulk Operations

```typescript
// List all settings for a scope
await settingsService.list({ scope: "store:str_01" })

// Delete all settings for a scope (e.g., on store deletion)
await settingsService.deleteByScope("store:str_01")
```

`deleteByScope` is used as a cleanup hook when a store or user entity is deleted.

## 9. Plugin Integration Pattern

```typescript
// In a custom plugin's onApplicationBootstrap
const settingsService = container.resolve("settingsModuleService")
await settingsService.set("acme_plugin.webhook_url", "https://...", {
  scope: "global",
})
```

## 10. Error Handling

| Scenario                  | Behaviour                                      |
|---------------------------|------------------------------------------------|
| Key not found at any scope | Returns `undefined`; no error thrown          |
| Invalid scope format      | Throws `MedusaError.Types.INVALID_DATA`        |
| Value exceeds size limit  | Throws `MedusaError.Types.INVALID_DATA` (>64KB)|

## 11. Configuration Options

The Settings module has no required configuration. It is registered as a core module automatically.

| Option           | Type   | Default | Description                 |
|------------------|--------|---------|-----------------------------|
| `max_value_size` | number | `65536` | Max JSON value size in bytes |
