# Software Design Document — RBAC Module

## 1. Purpose & Scope

This document describes the internal design of the Medusa RBAC (Role-Based Access Control) module (v2.15.4). It covers the data model, permission evaluation algorithm, middleware integration, role lifecycle, and the user-role link mechanism.

## 2. Architecture Overview

The RBAC module is structured in four layers:

1. **Data layer** — `Role` and `Permission` entities with a many-to-many join.
2. **Service layer** — `RbacModuleService` extending `MedusaService`, exposing CRUD operations.
3. **Link layer** — `UserRoleLink` (via Link Modules) mapping users to roles.
4. **Middleware layer** — `rbacMiddleware` evaluated on every admin HTTP request.

```
HTTP Request
  ↓
Auth Middleware (resolves user identity)
  ↓
RBAC Middleware
  ├─ query UserRoleLink → roles
  ├─ flatten permissions from all roles
  └─ evaluate(resource, action) → allow | 403
```

## 3. Data Model

### 3.1 Role

| Field         | Type      | Description                         |
|---------------|-----------|-------------------------------------|
| `id`          | string    | Unique ID (`role_*`)                |
| `name`        | string    | Display name                        |
| `description` | string    | Optional description                |
| `is_system`   | boolean   | True for built-in roles             |
| `permissions` | Permission[] | Associated permission records   |
| `created_at`  | timestamp | Creation timestamp                  |
| `updated_at`  | timestamp | Last update timestamp               |
| `deleted_at`  | timestamp | Soft-delete timestamp               |

### 3.2 Permission

| Field       | Type   | Description                              |
|-------------|--------|------------------------------------------|
| `id`        | string | Unique ID (`perm_*`)                     |
| `role_id`   | string | FK to role                               |
| `resource`  | string | Resource name (e.g., `order`, `product`) |
| `action`    | string | Action (e.g., `read`, `write`, `delete`) |

### 3.3 UserRoleLink (via Link Modules)

Stored in the link pivot table with columns `user_id` and `role_id`. No direct FK across module boundaries.

## 4. Permission Evaluation Algorithm

```typescript
function canAccess(userRoles: Role[], resource: string, action: string): boolean {
  for (const role of userRoles) {
    for (const perm of role.permissions) {
      const resourceMatch = perm.resource === "*" || perm.resource === resource
      const actionMatch  = perm.action === "*"  || perm.action === action
      if (resourceMatch && actionMatch) return true
    }
  }
  return false
}
```

Wildcard `*` is supported for both resource and action. Roles are evaluated additively (union).

## 5. Middleware Integration

`rbacMiddleware` is registered in the Medusa admin router after the auth middleware. It:

1. Reads `req.auth_context.actor_id` (set by auth middleware).
2. Resolves user roles via `query.graph({ entity: "user", fields: ["roles.*"] })`.
3. Extracts `resource` and `action` from the route definition metadata.
4. Calls `canAccess(roles, resource, action)`.
5. Returns `403 Forbidden` with a `MedusaError` if access is denied.

Route definitions annotate each endpoint with `{ rbac: { resource: "order", action: "read" } }` in the route's `config` export.

## 6. Role Lifecycle

### Seeding Built-in Roles

Built-in roles (`ADMIN`, `DEVELOPER`, `ANALYST`, `MANAGER`, `SUPPORT`) are created during `db:migrate` via seed data. They have `is_system: true` and cannot be deleted — only their permission set can be modified by operators.

### Custom Role Creation

```typescript
await rbacService.createRoles([{
  name: "Warehouse Staff",
  permissions: [
    { resource: "inventory", action: "read" },
    { resource: "inventory", action: "write" },
    { resource: "order",     action: "read" },
  ],
}])
```

## 7. User-Role Assignment

Role assignment is managed through the `UserRoleLink`:

```typescript
// POST /admin/users/:id/roles
await linkService.create({
  [Modules.USER]: { user_id: userId },
  [Modules.RBAC]: { role_id: roleId },
})
```

A user with no roles assigned is treated as having zero permissions. In `strict_mode: true`, this results in `403` for all resources.

## 8. Error Handling

| Scenario                      | Response                              |
|-------------------------------|---------------------------------------|
| No roles assigned             | 403 (strict mode) / pass-through      |
| Insufficient permission       | 403 with resource and action in body  |
| Role not found (assign)       | 404 `Role with id X was not found`    |
| Delete system role            | 400 `Cannot delete a system role`     |

## 9. Performance Considerations

Role lookups occur on every admin request. To minimise latency:
- Resolved roles are cached in the request context (scoped per request).
- The Query helper batches role + permission retrieval in a single SQL join.
- For high-throughput deployments, a short-lived Redis cache (TTL 30s) on `user:{id}:roles` is recommended.

## 10. Configuration Options

| Option       | Type    | Default | Description                                    |
|--------------|---------|---------|------------------------------------------------|
| `strict_mode`| boolean | `true`  | Deny access if user has no assigned roles      |
| `seed_roles` | boolean | `true`  | Seed built-in roles on database migration      |
