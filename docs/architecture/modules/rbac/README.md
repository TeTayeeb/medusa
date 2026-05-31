# RBAC Module

## Overview

The Role-Based Access Control (RBAC) module provides a structured permission system for the Medusa admin interface. It allows platform operators to define granular access policies by associating users with roles and roles with resource-level permissions.

The module integrates with the admin HTTP middleware layer, which evaluates role permissions on every authenticated request before allowing access to protected resources.

## Key Features

- **Role management**: Create, update, and delete named roles via `/admin/roles`.
- **Permission model**: Permissions are expressed as `resource + action` tuples (e.g., `order:read`, `product:write`).
- **Built-in roles**: `ADMIN`, `DEVELOPER`, `ANALYST`, `MANAGER`, `SUPPORT` ship as seeds; all are fully customisable.
- **Custom roles**: Operators can define domain-specific roles with arbitrary permission sets.
- **User-role assignment**: A user can hold multiple roles; permissions are union-merged.
- **Middleware enforcement**: Every admin API request passes through `rbacMiddleware`, which resolves the authenticated user's effective permissions before routing.

## Built-in Role Permissions (defaults)

| Role        | Resources                                   | Actions         |
|-------------|---------------------------------------------|-----------------|
| ADMIN       | *                                           | *               |
| DEVELOPER   | api-keys, workflows, plugins                | read, write     |
| ANALYST     | orders, products, analytics, reports        | read            |
| MANAGER     | orders, products, discounts, customers      | read, write     |
| SUPPORT     | orders, customers, returns, claims          | read, write     |

## API Surface

| Method | Path                          | Description                   |
|--------|-------------------------------|-------------------------------|
| GET    | `/admin/roles`                | List all roles                |
| POST   | `/admin/roles`                | Create a role                 |
| GET    | `/admin/roles/:id`            | Get role details              |
| PUT    | `/admin/roles/:id`            | Update role permissions       |
| DELETE | `/admin/roles/:id`            | Delete a role                 |
| POST   | `/admin/users/:id/roles`      | Assign roles to a user        |
| DELETE | `/admin/users/:id/roles/:rid` | Remove a role from a user     |

## Permission Check Flow

```
Request → Auth Middleware → RBAC Middleware
  → resolve user roles → merge permissions
  → evaluate(resource, action) → allow / 403
```

## Module Registration

```typescript
// medusa-config.ts
{
  resolve: "@medusajs/rbac",
  options: {
    strict_mode: true,   // deny if no role assigned
  },
}
```

## Dependencies

| Dependency              | Purpose                              |
|-------------------------|--------------------------------------|
| `@medusajs/framework`   | Container, middleware pipeline       |
| Auth module             | Resolves authenticated user identity |
| Link modules            | `user ↔ role` pivot relationship     |
| User module             | User entity lookup                   |

## Related Modules

- **Auth** – supplies the authenticated actor identity.
- **Link Modules** – stores the `user ↔ role` many-to-many relationship.
- **Settings** – may store default role for new users.
