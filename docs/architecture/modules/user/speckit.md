# Specification — User Module (SpecKit)

## 1. Module Identity

| Attribute       | Value                                         |
|-----------------|-----------------------------------------------|
| Module ID       | `Modules.USER`                                |
| Package         | `@medusajs/user`                              |
| Medusa Version  | 2.15.4                                        |
| Type            | Identity Management (profile + invite)        |
| Database tables | `user`, `invite`                              |
| API surface     | Admin only                                    |

## 2. Functional Requirements

### FR-USR-01: Create User
- **Given** a unique email address and optional profile fields
- **When** a user is created (via `createUsersWorkflow` or directly)
- **Then** a new User record is persisted and returned

### FR-USR-02: Update User Profile
- **Given** an existing User ID
- **When** `POST /admin/users/:id` is called with updated fields
- **Then** profile fields (name, avatar, metadata) are updated; email changes are validated for uniqueness

### FR-USR-03: Get Authenticated User
- **Given** a valid admin JWT
- **When** `GET /admin/users/me` is called
- **Then** return the User record associated with the JWT's actor identity

### FR-USR-04: Delete User
- **Given** an existing User ID
- **When** `DELETE /admin/users/:id` is called
- **Then** the user is soft-deleted; associated AuthIdentity link is removed

### FR-USR-05: Create Invite
- **Given** a unique email address not already registered
- **When** `POST /admin/invites` is called
- **Then** an Invite record is created with a cryptographically random token and 7-day expiry; `inviteCreated` hook is emitted

### FR-USR-06: Accept Invite
- **Given** a valid, non-expired, non-accepted invite token
- **When** `POST /admin/invites/accept` is called with token, name, and credentials
- **Then** a User record is created, an AuthIdentity is linked, and the Invite is marked accepted

### FR-USR-07: Reject Expired Token
- **Given** an invite token past its `expires_at` timestamp
- **When** `POST /admin/invites/accept` is called
- **Then** return 401 with error `MedusaError.Types.NOT_ALLOWED`

### FR-USR-08: Resend Invite
- **Given** an existing, non-accepted invite
- **When** `POST /admin/invites/:id/resend` is called
- **Then** a new token is generated, `expires_at` is reset, and the `inviteCreated` hook is re-emitted

### FR-USR-09: Revoke Invite
- **Given** an existing invite
- **When** `DELETE /admin/invites/:id` is called
- **Then** the invite is soft-deleted and the token becomes invalid

### FR-USR-10: List Users
- **Given** an authenticated admin
- **When** `GET /admin/users` is called
- **Then** return paginated list of non-deleted admin users

## 3. Non-Functional Requirements

| ID          | Requirement                    | Target                                         |
|-------------|--------------------------------|------------------------------------------------|
| NFR-USR-01  | Token entropy                  | ≥ 256 bits (32 bytes via `crypto.randomBytes`) |
| NFR-USR-02  | Token TTL                      | Default 7 days; configurable                   |
| NFR-USR-03  | Email uniqueness               | Enforced at database level (unique index)      |
| NFR-USR-04  | Accept invite atomicity        | User creation + auth link in one transaction   |
| NFR-USR-05  | Audit trail                    | Soft delete; deleted_at preserved              |

## 4. Interface Specification

### POST `/admin/invites`

| Attribute     | Value                                         |
|---------------|-----------------------------------------------|
| Auth required | Yes (Admin JWT)                               |
| Body          | `{ email: string, metadata?: object }`        |
| Response 200  | `{ invite: InviteDTO }`                       |
| Response 400  | Email already registered or invite exists     |

### POST `/admin/invites/accept`

| Attribute     | Value                                                                                          |
|---------------|------------------------------------------------------------------------------------------------|
| Auth required | No (public endpoint)                                                                           |
| Body          | `{ token: string, first_name?: string, last_name?: string, ...auth_data }`                   |
| Response 200  | `{ user: UserDTO, token: string }` (JWT for immediate login)                                  |
| Response 401  | Token expired, not found, or already accepted                                                  |

## 5. Data Contracts

### UserDTO

```ts
type UserDTO = {
  id: string
  first_name: string | null
  last_name: string | null
  email: string
  avatar_url: string | null
  metadata: Record<string, unknown> | null
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### InviteDTO

```ts
type InviteDTO = {
  id: string
  email: string
  accepted: boolean
  token: string
  expires_at: Date
  metadata: Record<string, unknown> | null
  created_at: Date
  updated_at: Date
  deleted_at: Date | null
}
```

### CreateInviteDTO

```ts
type CreateInviteDTO = {
  email: string
  metadata?: Record<string, unknown>
}
```

## 6. Validation Rules

| Field               | Rule                                                             |
|---------------------|------------------------------------------------------------------|
| `email`             | Valid email format; unique in `user` and `invite` tables         |
| `token` (accept)    | Must match existing invite; not expired; not previously accepted |
| `expires_at`        | Always set by server (not client-provided)                       |
| User `id` on delete | Cannot delete currently authenticated user (self-delete guard)   |

## 7. Edge Cases

| Case                                         | Expected Behaviour                                              |
|----------------------------------------------|-----------------------------------------------------------------|
| Accept invite with already-registered email  | Rejected; user creation fails with 409/400                     |
| Resend an already-accepted invite            | Rejected; returns `MedusaError.Types.NOT_ALLOWED`              |
| Accept invite exactly at `expires_at`        | Rejected (boundary: `expires_at < now()` check)                |
| Delete user with no associated AuthIdentity  | Allowed; link removal step is a no-op                          |
| Admin creates user without sending invite    | `createUsersWorkflow` can create user directly (no invite required) |
| Invite to existing user email                | Rejected at creation; unique constraint on invite.email         |

## 8. Module Boundaries

| In Scope                                  | Out of Scope                                          |
|-------------------------------------------|-------------------------------------------------------|
| User profile data (name, email, avatar)   | Password storage and validation (Auth module)         |
| Invite lifecycle (create, accept, revoke) | Session management and JWT issuance (Auth module)     |
| Token generation and expiry               | Role-based access control (RBAC module)               |
| Admin-only API                            | Customer/storefront accounts (Customer module)        |

## 9. Acceptance Criteria Summary

- [ ] `POST /admin/invites` creates invite, returns token in response
- [ ] Invite email is sent (Notification module subscriber fires)
- [ ] `POST /admin/invites/accept` with valid token creates User + AuthIdentity + returns JWT
- [ ] `POST /admin/invites/accept` with expired token returns 401
- [ ] `POST /admin/invites/accept` with already-accepted token returns 401
- [ ] `GET /admin/users/me` returns the authenticated user's profile
- [ ] `DELETE /admin/users/:id` soft-deletes; user not returned in subsequent list
- [ ] Two invites to same email are rejected on the second one
