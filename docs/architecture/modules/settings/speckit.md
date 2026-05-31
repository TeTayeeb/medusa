# SpecKit — Settings Module

## Module Identity

| Attribute      | Value                                     |
|----------------|-------------------------------------------|
| Module Name    | `@medusajs/settings`                      |
| Version        | 2.15.4                                    |
| Module Key     | `settings`                                |
| Type           | Infrastructure / Configuration            |
| Database Tables| `setting`                                 |
| Event Emitter  | No                                        |
| Event Consumer | No                                        |

---

## Functional Specifications

### SPEC-SETTINGS-001: Key-Value Storage
**Description**: The module MUST store arbitrary JSON values under namespaced keys. Values may be any valid JSON type (string, number, boolean, object, array).  
**Acceptance**: `set("foo.bar", { nested: true })` stores successfully; `get("foo.bar")` returns `{ nested: true }`.

### SPEC-SETTINGS-002: Scope Hierarchy Resolution
**Description**: `get()` MUST accept a `fallback` scope chain. Values are resolved from the most specific scope (first in chain) to the least specific (last). Returns the first non-null value found.  
**Acceptance**: User scope has value → user value returned. User scope empty, store scope has value → store value returned. No scope has value → `undefined` returned.

### SPEC-SETTINGS-003: Upsert Semantics
**Description**: `set()` MUST update an existing record for the same `(scope, key)` pair rather than creating a duplicate. If no record exists, a new one is created.  
**Acceptance**: Two calls to `set("k", "v1", scope)` then `set("k", "v2", scope)` → DB has one record with value `"v2"`.

### SPEC-SETTINGS-004: Scope Isolation
**Description**: Setting a value at one scope MUST NOT affect values at other scopes.  
**Acceptance**: `set("k", true, "user:u1")` does not change the value of `get("k", { scope: "global" })`.

### SPEC-SETTINGS-005: Delete Operation
**Description**: `delete()` MUST remove the setting record for the specified `(scope, key)` pair only. Other scopes' records for the same key are unaffected.  
**Acceptance**: Delete `user:u1` scope → `get()` with `fallback: ["store:s1"]` falls through to store scope value.

### SPEC-SETTINGS-006: Scope Bulk Delete
**Description**: `deleteByScope(scope)` MUST remove all setting records for the given scope. Intended for cleanup on entity deletion.  
**Acceptance**: `deleteByScope("store:str_01")` removes all settings scoped to that store; global settings unaffected.

### SPEC-SETTINGS-007: Value Size Limit
**Description**: Values exceeding 64KB in serialised JSON form MUST be rejected with `INVALID_DATA`.  
**Acceptance**: `set("k", largeObject)` where JSON > 64KB → throws MedusaError `INVALID_DATA`.

### SPEC-SETTINGS-008: Invalid Scope Format
**Description**: Scope strings not matching `global`, `store:{id}`, or `user:{id}` patterns MUST be rejected.  
**Acceptance**: `set("k", "v", { scope: "invalid" })` → throws `INVALID_DATA`.

---

## Non-Functional Specifications

### SPEC-SETTINGS-NFR-001: Read Performance
**Description**: A `get()` call with 2-level fallback MUST complete in < 3ms under normal load.  
**Target**: Indexed queries on `(scope, key)`; < 3ms p99.

### SPEC-SETTINGS-NFR-002: No Required Config
**Description**: The module MUST be available with zero explicit configuration. It registers automatically as a core module.  
**Target**: `medusa start` works without any settings module config.

---

## API Contract

Settings are not exposed via a dedicated REST API. They are accessed programmatically through the module service and surfaced through other module APIs (e.g., `/admin/stores` for store-scoped settings).

### Module Service

```typescript
get(key: string, options: { scope: string; fallback?: string[] }): Promise<unknown>
set(key: string, value: unknown, options: { scope: string }): Promise<SettingDTO>
delete(key: string, options: { scope: string }): Promise<void>
list(options: { scope: string }): Promise<SettingDTO[]>
deleteByScope(scope: string): Promise<void>
```

---

## Configuration Schema

```typescript
interface SettingsModuleOptions {
  max_value_size?: number   // bytes; default: 65536 (64KB)
}
```

---

## Test Checklist

- [ ] `set()` → `get()` returns correct value
- [ ] Scope hierarchy: user overrides store overrides global
- [ ] `set()` upserts; no duplicate records
- [ ] `delete()` removes only specified scope's record
- [ ] `deleteByScope()` removes all records for scope
- [ ] Value > 64KB rejected with INVALID_DATA
- [ ] Invalid scope string rejected with INVALID_DATA
- [ ] `get()` returns `undefined` when no scope has value

---

## Dependencies & Interfaces

| Dependency     | Interface Used              | Direction |
|----------------|-----------------------------|-----------|
| Store module   | `store.id` for scope prefix | Inbound   |
| User module    | `user.id` for scope prefix  | Inbound   |
| Analytics module | `get("analytics.opt_out")` | Outbound |
| Admin API (Store) | Proxies store-scoped settings | Bridge |
