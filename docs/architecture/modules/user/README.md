# User Module

The User module (`@medusajs/user`) manages admin user accounts and invitation flows in Medusa. It stores user profile information and handles token-based invitations for onboarding new team members. Authentication credential storage is explicitly delegated to the Auth module — the User module holds no passwords.

## Purpose

Separating user identity from authentication credentials follows the principle of separation of concerns. The User module owns profile data and access invitation logic; the Auth module owns authentication mechanisms (password hashing, OAuth tokens, API keys). This makes it straightforward to support multiple authentication methods per user without modifying the User module.

## Key Features

- **Admin User Accounts** — Stores first name, last name, email, and avatar for each admin user.
- **Invite Flow** — Generates time-limited, token-based invitations that allow new users to register and join the admin team.
- **No Password Storage** — Credentials are stored in the Auth module's `AuthIdentity` entity, not here.
- **Role Association** — User roles are managed via the RBAC module link, keeping authorization logic separate.
- **Metadata Support** — Free-form JSON metadata field for custom attributes.
- **Soft Delete** — Users can be deactivated without hard-deleting audit trail records.

## Entities

| Entity   | Key Fields                                                                                    |
|----------|-----------------------------------------------------------------------------------------------|
| `User`   | `id`, `first_name`, `last_name`, `email`, `avatar_url`, `metadata`                          |
| `Invite` | `id`, `email`, `accepted`, `token`, `expires_at`, `metadata`                                |

## Admin API

| Method | Endpoint                  | Description                           |
|--------|---------------------------|---------------------------------------|
| GET    | `/admin/users`            | List admin users                      |
| GET    | `/admin/users/me`         | Get the authenticated user's profile  |
| GET    | `/admin/users/:id`        | Retrieve a user by ID                 |
| POST   | `/admin/users/:id`        | Update a user profile                 |
| DELETE | `/admin/users/:id`        | Delete a user                         |
| GET    | `/admin/invites`          | List invites                          |
| POST   | `/admin/invites`          | Create (send) an invite               |
| POST   | `/admin/invites/accept`   | Accept an invite (create user)        |
| POST   | `/admin/invites/:id/resend` | Resend an invite                    |
| DELETE | `/admin/invites/:id`      | Revoke an invite                      |

## Store API

No Store API endpoints. This module is exclusively admin-facing.

## Module Identifier

```ts
import { Modules } from "@medusajs/framework/utils"
// Modules.USER
```

## Service Usage

```ts
const userService = container.resolve(Modules.USER)

// Create a user
const user = await userService.createUsers({
  email: "admin@example.com",
  first_name: "Jane",
  last_name: "Doe",
})

// Create an invite
const invite = await userService.createInvites({
  email: "newmember@example.com",
})

// Retrieve invite by token (during acceptance)
const invite = await userService.retrieveInviteByToken(token)

// Mark invite as accepted
await userService.updateInvites(invite.id, { accepted: true })
```

## Invite Flow

```
Admin          → POST /admin/invites         → Create Invite (token generated)
Email System   → Send invite email with token link
New User       → POST /admin/invites/accept  → Validate token, create User + AuthIdentity
```

## Module Links

| Link                  | Description                                              |
|-----------------------|----------------------------------------------------------|
| `user ↔ auth-identity`| Links a User to their Auth module credential record      |

## Related Modules

- **Auth Module** — Manages authentication credentials (passwords, OAuth) associated with user accounts.
- **API Key Module** — Admin users can have API keys for programmatic access.

## Version

Medusa v2.15.4
