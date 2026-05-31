# Architecture Documentation — User Module (arc42)

## 1. Introduction and Goals

The User module manages admin user profiles and invitation lifecycle for Medusa's back-office. It is designed with a strict boundary: it owns identity metadata (name, email, avatar) and invitation state, but explicitly delegates authentication to the Auth module. This separation supports multiple authentication strategies (password, OAuth, SSO) without requiring changes to the User module.

**Quality Goals:**

| Priority | Quality Goal   | Description                                                                   |
|----------|----------------|-------------------------------------------------------------------------------|
| 1        | Separation of Concerns | User profile data is cleanly separated from authentication credentials |
| 2        | Security       | Invite tokens have high entropy and enforced TTL                               |
| 3        | Auditability   | Soft delete preserves historical user records                                  |
| 4        | Simplicity     | Profile management is straightforward CRUD; complexity lives in Auth module   |

## 2. Constraints

- User email addresses must be unique across the `user` table.
- No password or credential data may be stored in the User module.
- Invite tokens must expire (default TTL: 7 days) and be validated before acceptance.
- Admin API routes must require authentication except for invite acceptance endpoints.

## 3. Context and Scope

```
External:
  [Admin Browser] ──CRUD──────► [Admin API /admin/users]
  [Admin Browser] ──invite────► [Admin API /admin/invites]
  [Email System]  ◄─── notification triggered by inviteCreated hook

Internal:
  [User Module]   ──linked to──► [Auth Module] (AuthIdentity ↔ User)
  [Auth Module]   ──resolves actor──► [User Module] (on JWT validation)
  [Notification Module] ◄── subscribe to inviteCreated event
```

## 4. Solution Strategy

| Challenge                               | Strategy                                                          |
|-----------------------------------------|-------------------------------------------------------------------|
| Supporting multiple auth methods        | No credentials in User module; Auth module owns all auth identity |
| Secure invite flow                      | 256-bit random token; TTL-enforced; single-use (accepted flag)   |
| Linking user to auth credentials        | Module link: `UserModule.User ↔ AuthModule.AuthIdentity`         |
| Admin-only access                       | All endpoints require `actor_type: "user"` JWT claim             |

## 5. Building Block View

```
User Module
├── HTTP Layer
│   ├── Admin Routes: /admin/users (GET, POST, DELETE)
│   ├── Admin Routes: /admin/users/me (GET)
│   └── Admin Routes: /admin/invites (GET, POST, DELETE, /accept, /:id/resend)
│
├── Workflow Layer
│   ├── createUsersWorkflow
│   ├── updateUsersWorkflow
│   ├── deleteUsersWorkflow
│   ├── createInviteWorkflow        ← emits inviteCreated hook
│   ├── acceptInviteWorkflow        ← multi-step: validate + create user + link auth
│   └── refreshInviteTokenWorkflow
│
├── Service Layer
│   └── UserModuleService
│       ├── createUsers / updateUsers / deleteUsers
│       ├── listUsers / retrieveUser
│       ├── createInvites / updateInvites / deleteInvites
│       ├── listInvites / retrieveInvite
│       └── validateInviteToken / retrieveInviteByToken
│
└── Domain Model
    ├── User
    └── Invite
```

## 6. Runtime View

**Scenario A: Admin invites a new team member**

```
POST /admin/invites { email: "new@example.com" }
  → Auth middleware validates admin JWT
  → createInviteWorkflow.run({ email })
  → createInvitesStep: generate token (32 random bytes), set expires_at = now + 7d
  → Persist Invite record (status: pending)
  → inviteCreated hook emitted
  → Notification subscriber: createNotifications({ template: "invite", to: email })
  → SendGrid provider sends email with invite link
  → Response: { invite: InviteDTO }
```

**Scenario B: New user accepts invite**

```
POST /admin/invites/accept { token: "abc123...", first_name: "Jane", password: "..." }
  → acceptInviteWorkflow.run({ token, user_data, auth_data })
  → validateInviteTokenStep: retrieve invite by token, check expires_at, check !accepted
  → createUsersStep: create User record
  → createAuthIdentityStep: create AuthIdentity in Auth module with hashed password
  → createRemoteLinkStep: link User ↔ AuthIdentity
  → markInviteAcceptedStep: update Invite.accepted = true
  → Response: JWT token for immediate login
```

## 7. Deployment View

Single Medusa process. Uses two database tables (`user`, `invite`) in the shared PostgreSQL instance. No external service dependencies (token generation is local via `crypto.randomBytes`).

## 8. Cross-Cutting Concerns

| Concern        | Approach                                                                      |
|----------------|-------------------------------------------------------------------------------|
| Authentication | All routes except `POST /admin/invites/accept` require admin JWT             |
| Token security | `crypto.randomBytes(32)` for 256-bit entropy; stored as hex string            |
| Soft delete    | `deleted_at` on both User and Invite; historical records preserved            |
| Email uniqueness | Database unique constraint; race condition handled at DB level               |
| Transactions   | `acceptInviteWorkflow` executes all steps within a distributed transaction    |

## 9. Design Decisions

| ID  | Decision                            | Rationale                                                                     |
|-----|-------------------------------------|-------------------------------------------------------------------------------|
| D1  | No password storage in User module  | Clean separation; Auth module can be replaced/extended independently          |
| D2  | Module link for User ↔ AuthIdentity | Loose coupling; Auth module does not import from User module                  |
| D3  | Invite.accepted = true on use       | Prevents token replay attacks; token becomes invalid after acceptance         |
| D4  | 7-day invite TTL                    | Balance between convenience and security; configurable via module options     |

## 10. Risks and Technical Debt

| Risk                                  | Mitigation                                             |
|---------------------------------------|--------------------------------------------------------|
| Invite token interception             | HTTPS enforced; tokens are one-time use                |
| Auth module unavailable during accept | Workflow rolls back user creation via compensation     |
| Admin deletes user with active session| Session invalidated on next request via JWT validation |
