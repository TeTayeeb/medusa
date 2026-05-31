# Software Design Document — User Module

## 1. Overview

The User module manages admin user profiles and invitation lifecycle in Medusa. It is a deliberately narrow module: it stores identity-adjacent data (name, email, avatar) and invite tokens, but intentionally contains no authentication logic, no password hashing, and no session management. Those concerns are owned by the Auth module. This separation allows the platform to support multiple authentication strategies (JWT, OAuth, SSO) without modifying the User module.

## 2. Goals and Non-Goals

**Goals:**
- Maintain a profile record for each admin user (name, email, avatar, metadata).
- Implement a token-based invite flow for onboarding new team members.
- Provide Admin API endpoints for user and invite management.
- Link user records to auth identities via the module link layer.

**Non-Goals:**
- Password storage or validation (Auth module responsibility).
- Session management or JWT issuance (Auth module responsibility).
- Role-based access control (RBAC module responsibility).
- Storefront customer accounts (Customer module responsibility).

## 3. Data Model

### 3.1 User Entity

```ts
@Entity()
export class User extends BaseEntity {
  @PrimaryKey()
  id: string  // ULID

  @Property({ nullable: true })
  first_name?: string

  @Property({ nullable: true })
  last_name?: string

  @Property({ unique: true })
  email: string

  @Property({ nullable: true })
  avatar_url?: string

  @Property({ nullable: true, type: "jsonb" })
  metadata?: Record<string, unknown>

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.2 Invite Entity

```ts
@Entity()
export class Invite extends BaseEntity {
  @PrimaryKey()
  id: string

  @Property({ unique: true })
  email: string

  @Property({ default: false })
  accepted: boolean

  @Property()
  token: string  // Cryptographically random, used in acceptance URL

  @Property()
  expires_at: Date

  @Property({ nullable: true, type: "jsonb" })
  metadata?: Record<string, unknown>

  @Property({ onCreate: () => new Date() })
  created_at: Date

  @Property({ onUpdate: () => new Date() })
  updated_at: Date

  @Property({ nullable: true })
  deleted_at?: Date
}
```

### 3.3 Database Tables

| Table    | Primary Key | Notable Indexes                  |
|----------|-------------|----------------------------------|
| `user`   | `id` (ULID) | `email` (unique), `deleted_at`   |
| `invite` | `id` (ULID) | `email` (unique), `token`, `expires_at` |

## 4. Service Interface

```ts
interface IUserModuleService {
  createUsers(data: CreateUserDTO[], sharedContext?: Context): Promise<UserDTO[]>
  updateUsers(data: UpdateUserDTO[], sharedContext?: Context): Promise<UserDTO[]>
  deleteUsers(ids: string[], sharedContext?: Context): Promise<void>
  retrieveUser(id: string, config?: FindConfig<UserDTO>, sharedContext?: Context): Promise<UserDTO>
  listUsers(filters?: FilterableUserProps, config?: FindConfig<UserDTO>, sharedContext?: Context): Promise<UserDTO[]>

  createInvites(data: CreateInviteDTO[], sharedContext?: Context): Promise<InviteDTO[]>
  updateInvites(data: UpdateInviteDTO[], sharedContext?: Context): Promise<InviteDTO[]>
  deleteInvites(ids: string[], sharedContext?: Context): Promise<void>
  retrieveInvite(id: string, config?: FindConfig<InviteDTO>, sharedContext?: Context): Promise<InviteDTO>
  listInvites(filters?: FilterableInviteProps, config?: FindConfig<InviteDTO>, sharedContext?: Context): Promise<InviteDTO[]>
  retrieveInviteByToken(token: string, sharedContext?: Context): Promise<InviteDTO>
  validateInviteToken(token: string, sharedContext?: Context): Promise<InviteDTO>
}
```

## 5. Invite Lifecycle

```
State machine: pending → accepted | expired | revoked

pending:   Invite created, token generated, email dispatched (via Notification module)
accepted:  User completes registration; Invite.accepted = true; User record created
expired:   Invite.expires_at < now(); retrieveInviteByToken throws NOT_FOUND
revoked:   Admin calls DELETE /admin/invites/:id; soft-deleted
```

Token generation uses `crypto.randomBytes(32).toString("hex")` for 256-bit entropy. Expiry defaults to 7 days from creation.

## 6. Module Architecture

```
@medusajs/user
├── src/
│   ├── models/
│   │   ├── user.ts
│   │   └── invite.ts
│   ├── services/
│   │   └── user-module-service.ts
│   ├── migrations/
│   └── index.ts
```

## 7. Workflows

| Workflow                   | Steps                                              |
|----------------------------|----------------------------------------------------|
| `createUsersWorkflow`      | `createUsersStep`                                  |
| `updateUsersWorkflow`      | `updateUsersStep`                                  |
| `deleteUsersWorkflow`      | `deleteUsersStep`                                  |
| `createInviteWorkflow`     | `createInvitesStep`, emit `inviteCreated` hook     |
| `acceptInviteWorkflow`     | `validateInviteTokenStep`, `createUsersStep`, `createAuthIdentityStep`, `markInviteAcceptedStep` |
| `refreshInviteTokenWorkflow` | `refreshInviteTokenStep`                         |

## 8. Module Link: User ↔ AuthIdentity

The link between a `User` and an `AuthIdentity` is managed outside both modules:

```
UserModule.User ──< link >── AuthModule.AuthIdentity
```

When a user logs in, the Auth module resolves the `AuthIdentity` and follows the link to find the associated `User.id`, which is embedded in the JWT token as the actor identity.

## 9. Security Considerations

| Concern              | Approach                                              |
|----------------------|-------------------------------------------------------|
| Token entropy        | 256-bit random token via `crypto.randomBytes`         |
| Token expiry         | Hard-coded TTL (default 7 days), checked on retrieval |
| Email uniqueness     | Database unique constraint on `User.email` and `Invite.email` |
| Soft delete          | Preserves audit trail; email freed only after hard delete |
| Admin-only API       | All endpoints require `actor_type: "user"` JWT claim  |

## 10. Error Handling

| Condition                          | Error Type                        |
|------------------------------------|-----------------------------------|
| User not found                     | `MedusaError.Types.NOT_FOUND`     |
| Invite not found                   | `MedusaError.Types.NOT_FOUND`     |
| Expired invite token               | `MedusaError.Types.NOT_ALLOWED`   |
| Already-accepted invite            | `MedusaError.Types.NOT_ALLOWED`   |
| Duplicate email on invite/user     | Database constraint → `INVALID_DATA` |
