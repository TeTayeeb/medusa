# arc42 Architecture Document — RBAC Module

## 1. Introduction and Goals

### 1.1 Requirements Overview
The RBAC module must enforce resource-level access control on all admin API routes, support custom role definitions, integrate with the existing user identity system, and remain performant under concurrent admin usage.

### 1.2 Quality Goals

| Priority | Quality Goal    | Scenario                                                        |
|----------|-----------------|-----------------------------------------------------------------|
| 1        | Security        | A user without `order:write` cannot cancel orders via the API   |
| 2        | Correctness     | Role changes take effect on the next request; no stale cache    |
| 3        | Flexibility     | A new role with custom permissions can be created without code changes |
| 4        | Performance     | Permission check adds < 5ms to every admin request              |

### 1.3 Stakeholders

| Role              | Expectation                                                  |
|-------------------|--------------------------------------------------------------|
| Platform operator | Fine-grained control over which admin users can do what      |
| Developer         | Simple, declarative role/permission definitions              |
| Security auditor  | All access decisions logged; no privilege escalation paths   |

---

## 2. Architecture Constraints

- Must not introduce hard coupling to the User module (must use Link Modules for user-role association).
- System roles (`ADMIN`, etc.) must always be present; operators cannot delete them.
- Role resolution must complete within a single HTTP request lifecycle.

---

## 3. System Scope and Context

```
┌──────────────────────────────────────────────┐
│               Admin HTTP Layer               │
│  Auth Middleware → RBAC Middleware → Route   │
└──────────────────┬───────────────────────────┘
                   │
   ┌───────────────▼──────────────────────────┐
   │            RBAC Module                   │
   │  RbacModuleService                       │
   │  ├─ Role CRUD                            │
   │  └─ Permission management                │
   └──────────────────────────────────────────┘
                   │ UserRoleLink
   ┌───────────────▼──────────────────────────┐
   │            User Module                   │
   │  User entity (actor identity)            │
   └──────────────────────────────────────────┘
```

---

## 4. Solution Strategy

- Roles and permissions are stored in the RBAC module's own schema; no cross-module FKs.
- `UserRoleLink` (a Link Module pivot table) provides the user-role association.
- The middleware resolves permissions at request time using the Query helper.
- Wildcard permissions (`*:*`) enable super-admin bypass without special-casing in the evaluation algorithm.

---

## 5. Building Block View

### Level 1

```
RbacModule
  ├── RbacModuleService        (MedusaService<{Role, Permission}>)
  ├── Role                     (entity)
  ├── Permission               (entity)
  ├── UserRoleLink             (link module pivot)
  └── rbacMiddleware           (HTTP middleware)
```

### Level 2 — rbacMiddleware

```
rbacMiddleware
  ├── resolveActor(req)        (reads req.auth_context)
  ├── fetchRoles(userId)       (Query helper call)
  ├── flattenPermissions(roles)
  └── canAccess(resource, action) → allow | throw 403
```

---

## 6. Runtime View

### Scenario: Analyst Tries to Cancel an Order

1. `DELETE /admin/orders/:id` received.
2. Auth middleware sets `req.auth_context.actor_id = usr_01`.
3. RBAC middleware resolves `usr_01`'s roles: `[ANALYST]`.
4. ANALYST permissions: `{ order: read, product: read, ... }` — no `order:delete`.
5. Route metadata: `{ rbac: { resource: "order", action: "delete" } }`.
6. `canAccess("order", "delete")` returns `false`.
7. Response: `403 Forbidden — Insufficient permissions for order:delete`.

---

## 7. Deployment View

The RBAC module runs in-process within the Medusa server. The `role` and `permission` tables reside in the same PostgreSQL database as other Medusa modules. No separate service is required.

---

## 8. Cross-Cutting Concerns

### Audit Logging
Permission denials are logged at `warn` level with `{ userId, resource, action, route }`. Access grants are logged at `debug` level.

### Cache Strategy
Role data is resolved fresh on each request. For high-traffic deployments (> 100 req/s), a per-request cache keyed on `userId` prevents redundant DB queries within the same request lifecycle.

### Seeding
Built-in roles are seeded via the migration runner. Idempotent `upsert` semantics prevent duplicates on re-migration.

---

## 9. Architecture Decisions

| ID  | Decision                                      | Rationale                                                 |
|-----|-----------------------------------------------|-----------------------------------------------------------|
| AD1 | Permissions stored per role, not per user     | Roles are reusable; per-user permissions do not scale     |
| AD2 | Link module for user-role association         | Preserves module isolation; no cross-schema FK            |
| AD3 | Union (additive) permission merge across roles | Matches the principle of least surprise for multi-role users |
| AD4 | System roles flagged `is_system: true`        | Prevents accidental deletion of critical roles            |

---

## 10. Quality Scenarios

| Quality     | Scenario                                            | Measure                            |
|-------------|-----------------------------------------------------|------------------------------------|
| Security    | Non-admin creates product via API                   | 403 returned; no DB write occurs   |
| Correctness | Role permission removed → next request denied       | No cache staleness; fresh resolve  |
| Performance | 50 concurrent admin requests with role resolution   | < 5ms p99 added per request        |
