# SpecKit — RBAC Module

## Module Identity

| Attribute      | Value                                     |
|----------------|-------------------------------------------|
| Module Name    | `@medusajs/rbac`                          |
| Version        | 2.15.4                                    |
| Module Key     | `rbac`                                    |
| Type           | Security / Access Control                 |
| Database Tables| `role`, `permission`                      |
| Event Emitter  | No                                        |
| Event Consumer | No                                        |

---

## Functional Specifications

### SPEC-RBAC-001: Role CRUD
**Description**: The module MUST expose full create, read, update, and delete operations for roles via `/admin/roles`.  
**Acceptance**: A role created via POST is retrievable via GET. Updated permissions are reflected immediately. Deleted role is no longer returned.

### SPEC-RBAC-002: System Role Protection
**Description**: Roles with `is_system: true` MUST NOT be deletable. Attempting to delete a system role MUST return `400 Bad Request`.  
**Acceptance**: `DELETE /admin/roles/{system_role_id}` → 400 with message `Cannot delete a system role`.

### SPEC-RBAC-003: Wildcard Permissions
**Description**: A permission with `resource: "*"` and/or `action: "*"` MUST match any resource and/or action respectively.  
**Acceptance**: ADMIN role (wildcard) passes `canAccess("order", "delete")` check.

### SPEC-RBAC-004: Additive Role Merge
**Description**: When a user has multiple roles, their effective permissions MUST be the union of all role permissions. No role can reduce permissions granted by another.  
**Acceptance**: User with ANALYST + MANAGER roles can write orders (MANAGER grants it) even though ANALYST does not.

### SPEC-RBAC-005: Access Denial on Insufficient Permission
**Description**: An admin request for a resource the user lacks permission on MUST return `403 Forbidden` with `{ resource, action }` in the error body.  
**Acceptance**: ANALYST user calls `DELETE /admin/orders/:id` → 403 with `{ resource: "order", action: "delete" }`.

### SPEC-RBAC-006: Strict Mode Behaviour
**Description**: With `strict_mode: true`, a user with no assigned roles MUST be denied access to all resources.  
**Acceptance**: New user with no roles → 403 on any admin route (except auth endpoints).

### SPEC-RBAC-007: User-Role Assignment
**Description**: Multiple roles MUST be assignable to a single user via `POST /admin/users/:id/roles`. Roles are stored in the `UserRoleLink` pivot.  
**Acceptance**: User assigned ANALYST and SUPPORT roles; both appear in `GET /admin/users/:id` role list.

### SPEC-RBAC-008: Role Seeding
**Description**: Built-in roles (ADMIN, DEVELOPER, ANALYST, MANAGER, SUPPORT) MUST be present after `db:migrate`. Seeding must be idempotent.  
**Acceptance**: Running migration twice does not create duplicate roles.

---

## Non-Functional Specifications

### SPEC-RBAC-NFR-001: Middleware Latency
**Description**: RBAC permission check MUST add < 5ms p99 latency to admin requests.  
**Target**: Measured under 50 concurrent admin requests; role resolution < 5ms.

### SPEC-RBAC-NFR-002: No Stale Permissions
**Description**: Permission changes MUST take effect on the next request. No TTL caching at the module level.  
**Target**: Role updated; next request reflects new permissions without restart.

---

## API Contract

### `GET /admin/roles`
**Response**: `200` — `{ roles: Role[], count, limit, offset }`

### `POST /admin/roles`
**Body**: `{ name: string, permissions: { resource: string, action: string }[] }`  
**Response**: `201` — `{ role: Role }`

### `PUT /admin/roles/:id`
**Body**: `{ name?: string, permissions?: Permission[] }`  
**Response**: `200` — `{ role: Role }`

### `DELETE /admin/roles/:id`
**Response**: `200` — `{ id, deleted: true }` or `400` if system role

### `POST /admin/users/:id/roles`
**Body**: `{ role_ids: string[] }`  
**Response**: `200` — `{ user: User }`

---

## Configuration Schema

```typescript
interface RbacModuleOptions {
  strict_mode?: boolean   // default: true — deny if no role assigned
  seed_roles?: boolean    // default: true — seed built-in roles on migrate
}
```

---

## Test Checklist

- [ ] ADMIN wildcard grants access to all resources
- [ ] ANALYST denied write on order
- [ ] User with no roles denied in strict mode
- [ ] User with no roles allowed in non-strict mode
- [ ] System role delete returns 400
- [ ] Role update immediately reflected in next request
- [ ] User assigned two roles: union of permissions applies
- [ ] Seed migration idempotent (no duplicates on re-run)

---

## Dependencies & Interfaces

| Dependency     | Interface Used                | Direction |
|----------------|-------------------------------|-----------|
| Auth module    | `req.auth_context.actor_id`   | Inbound   |
| Link Modules   | `UserRoleLink` pivot          | Outbound  |
| Query helper   | `query.graph` for role lookup | Outbound  |
| User module    | User entity (via link)        | Outbound  |
