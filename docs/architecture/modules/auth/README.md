# Auth Module

**Package:** `@medusajs/auth`  
**Version:** Medusa v2.15.4  
**Module Key:** `Modules.AUTH`

## Overview

The Auth Module provides a provider-agnostic authentication and identity management layer for Medusa v2. It decouples identity records from actor entities (Admin Users, Customers) by introducing the `AuthIdentity` concept — a stable identity record that may be linked to any actor type across the system. This design enables a single user to authenticate via multiple providers (email/password, GitHub, Google) while maintaining one canonical identity.

The module manages the authentication flow itself (credential validation, OAuth callbacks, MFA challenges) but delegates JWT issuance and session management to the Medusa HTTP framework layer. It does not store passwords in plaintext; the email/password provider (`@medusajs/auth-emailpass`) handles bcrypt hashing internally.

## Key Concepts

### AuthIdentity

`AuthIdentity` is the root identity entity. It is actor-type agnostic — the same `AuthIdentity` can represent an admin user or a customer depending on which actor record it is linked to via module links.

Key characteristics:
- Has a stable `id` (prefix `authid`) that serves as a stable reference across providers
- Contains `app_metadata` — a JSON blob for application-level metadata (roles, actor IDs, permissions)
- Owns multiple `ProviderIdentity` records — one per authentication provider
- Owns `AuthMfaFactor` records for multi-factor authentication setup
- Owns `AuthMfaRecoveryCode` records for MFA account recovery

### ProviderIdentity

`ProviderIdentity` represents the credentials or profile data for a specific authentication provider. Fields:
- `entity_id` — the user's identifier within that provider (email address, GitHub user ID, Google sub claim, etc.)
- `provider` — the provider ID string (e.g., `emailpass`, `github`, `google`)
- `user_metadata` — JSON data supplied or editable by the user (e.g., display name)
- `provider_metadata` — JSON data stored and managed by the auth provider (e.g., hashed password, OAuth tokens)

The combination `(entity_id, provider)` is unique — a given external identity maps to exactly one `ProviderIdentity`.

### Multi-Factor Authentication (MFA)

Since Medusa v2.12+, the Auth Module supports MFA through two supporting entities:

**AuthMfaFactor**  
Represents a configured MFA method (e.g., TOTP authenticator app). Fields:
- `provider` — the MFA provider (e.g., `totp`)
- `status` — lifecycle state: `pending` (enrolled but not verified), `enabled` (active), soft-deleted when removed
- `provider_metadata` — provider-specific data (e.g., TOTP secret, QR code URI)
- A unique constraint ensures at most one active factor per (identity, provider) combination

**AuthMfaRecoveryCode**  
One-time backup codes for MFA recovery. Stored as `code_hash` (bcrypt), so plaintext codes are never persisted. Unique per (identity, code_hash) combination.

### Auth Providers

Authentication logic is implemented in provider packages that implement the `AbstractAuthModuleProvider` interface:

| Provider Package | ID | Description |
|---|---|---|
| `@medusajs/auth-emailpass` | `emailpass` | Email + password with bcrypt hashing |
| `@medusajs/auth-github` | `github` | OAuth 2.0 via GitHub |
| `@medusajs/auth-google` | `google` | OAuth 2.0 via Google |

Custom providers can be created by extending `AbstractAuthModuleProvider` and implementing `authenticate()` and `validateCallback()`.

### Provider Interface

```typescript
abstract class AbstractAuthModuleProvider {
  abstract authenticate(
    data: Record<string, unknown>,
    authIdentityProviderService: AuthIdentityProviderService
  ): Promise<AuthenticationResponse>

  abstract validateCallback(
    data: Record<string, unknown>,
    authIdentityProviderService: AuthIdentityProviderService
  ): Promise<AuthenticationResponse>
}
```

`AuthenticationResponse` returns either a success (with `authIdentity`) or an error (with HTTP `status` code and `message`), along with a `location` redirect URL for OAuth flows.

## Authentication Flows

### Email/Password Flow
1. Client POSTs credentials to `/store/auth/emailpass` or `/admin/auth/emailpass`
2. Framework routes to `IAuthModuleService.authenticate(provider, providerData, authScope)`
3. `emailpass` provider verifies password hash against stored `provider_metadata`
4. On success, the framework issues a JWT containing `actor_id` and `actor_type` from `app_metadata`

### OAuth Flow (e.g., GitHub)
1. Client redirects to `/store/auth/github`
2. Framework generates OAuth authorization URL and redirects
3. Provider redirects back to `/store/auth/github/callback?code=...`
4. Framework calls `IAuthModuleService.validateCallback(provider, callbackData, authScope)`
5. Provider exchanges code for token, fetches user profile, upserts `ProviderIdentity`
6. Framework issues JWT on success

### MFA Flow
1. After initial credential verification, if an `enabled` MFA factor exists, authentication returns `requires_mfa: true`
2. Client submits MFA code to a dedicated endpoint
3. Framework verifies the code via the MFA provider
4. On success, the full JWT is issued

## Admin & Store API

| Endpoint | Description |
|---|---|
| `POST /store/auth/:provider` | Authenticate via provider (store) |
| `GET /store/auth/:provider/callback` | OAuth callback (store) |
| `POST /admin/auth/:provider` | Authenticate via provider (admin) |
| `GET /admin/auth/:provider/callback` | OAuth callback (admin) |
| `POST /admin/auth/token/refresh` | Refresh JWT token |
| `DELETE /admin/auth` | Logout (revoke session) |

## Module Links

- **Customer ↔ AuthIdentity** — links a Customer actor to an `AuthIdentity`
- **User (Admin) ↔ AuthIdentity** — links an Admin User to an `AuthIdentity`

The `actor_id` stored in the JWT's `app_metadata` identifies which Customer or User record is associated with the authenticated identity.
