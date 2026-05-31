# Software Design Document — Auth Module

**Module:** `@medusajs/auth`  
**Version:** Medusa v2.15.4  
**Status:** Production  
**Last Updated:** 2025

---

## 1. Purpose and Scope

This document describes the internal design, data model, service interface, provider abstraction, and MFA architecture of the Auth Module. The module manages authentication identity lifecycle: creation, provider-based verification, token exchange, and multi-factor authentication.

---

## 2. System Context

The Auth Module is invoked by API route handlers in both the Admin and Store API surfaces. It does not issue JWTs directly — that responsibility belongs to the HTTP framework layer. The module returns an `AuthenticationResponse` which the framework then uses to mint a JWT if authentication succeeds.

```
HTTP Request → API Route → IAuthModuleService.authenticate(provider, data, scope)
                        → AbstractAuthModuleProvider.authenticate(data, identityService)
                        → AuthenticationResponse { authIdentity, success, location? }
                        → Framework: mint JWT with actor_id, actor_type from app_metadata
```

---

## 3. Data Model

### 3.1 Entity Relationship Summary

| Entity | Table | PK Prefix | Notes |
|---|---|---|---|
| `AuthIdentity` | `auth_identity` | `authid_` | Root identity, actor-agnostic |
| `ProviderIdentity` | `provider_identity` | (none) | Per-provider credentials |
| `AuthMfaFactor` | `auth_mfa_factor` | `authmfa_` | MFA method enrollment |
| `AuthMfaRecoveryCode` | `auth_mfa_recovery_code` | `authmfarec_` | Backup recovery codes |

### 3.2 AuthIdentity Fields

```
AuthIdentity {
  id            : string (PK, prefix: authid)
  app_metadata  : json | null  -- { actor_id, actor_type, roles[], ... }
  created_at    : datetime
  updated_at    : datetime
  deleted_at    : datetime | null
}
```

`app_metadata` is the bridge between authentication and authorization. The framework reads `actor_id` and `actor_type` from this field to identify which Customer or User record the authenticated identity belongs to.

Cascade delete → `provider_identities`, `mfa_factors`, `mfa_recovery_codes`.

### 3.3 ProviderIdentity Fields

```
ProviderIdentity {
  id                : string (PK, no prefix — uses ulid)
  entity_id         : string   -- external user ID (email, GitHub ID, Google sub, etc.)
  provider          : string   -- provider ID (emailpass | github | google | ...)
  auth_identity_id  : FK → AuthIdentity
  user_metadata     : json | null   -- user-editable metadata
  provider_metadata : json | null   -- provider-managed (e.g., password hash, OAuth tokens)
  created_at        : datetime
  updated_at        : datetime
  deleted_at        : datetime | null
}
```

**Key index:**
- `IDX_provider_identity_provider_entity_id` — unique on `(entity_id, provider)` — ensures a given external identity maps to exactly one `ProviderIdentity`

The `provider_metadata` field is the provider's private storage. For `emailpass` this contains the bcrypt password hash. For OAuth providers it may contain the access token and refresh token. This field is never returned in API responses.

### 3.4 AuthMfaFactor Fields

```
AuthMfaFactor {
  id                : string (PK, prefix: authmfa)
  auth_identity_id  : FK → AuthIdentity
  provider          : string   -- e.g. "totp"
  status            : string   -- "pending" | "enabled"
  provider_metadata : json | null   -- e.g. TOTP secret, QR URI
  metadata          : json | null
  created_at        : datetime
  updated_at        : datetime
  deleted_at        : datetime | null
}
```

**Key indexes:**
- `IDX_auth_mfa_factor_auth_identity_id` — fast lookup of factors for an identity
- `IDX_auth_mfa_factor_auth_identity_provider_active` — unique on `(auth_identity_id, provider)` where `status IN ('pending', 'enabled')` — prevents duplicate active factors per provider

MFA factor lifecycle: `pending` → `enabled` (verified) → soft-deleted (removed).

### 3.5 AuthMfaRecoveryCode Fields

```
AuthMfaRecoveryCode {
  id               : string (PK, prefix: authmfarec)
  auth_identity_id : FK → AuthIdentity
  code_hash        : string   -- bcrypt hash of the recovery code
  created_at       : datetime
  deleted_at       : datetime | null
}
```

Plaintext codes are generated and returned to the user once. The module immediately hashes them. Recovery consumption soft-deletes the used code.

---

## 4. Service Layer

### 4.1 IAuthModuleService

```typescript
interface IAuthModuleService {
  // Authentication flows
  authenticate(
    provider: string,
    providerData: Record<string, unknown>,
    authScope: AuthScope
  ): Promise<AuthenticationResponse>

  validateCallback(
    provider: string,
    callbackData: Record<string, unknown>,
    authScope: AuthScope
  ): Promise<AuthenticationResponse>

  // Identity management
  createAuthIdentities(data, context?): Promise<AuthIdentityDTO[]>
  updateAuthIdentities(data, context?): Promise<AuthIdentityDTO[]>
  deleteAuthIdentities(ids, context?): Promise<void>
  retrieveAuthIdentity(id, config?, context?): Promise<AuthIdentityDTO>
  listAuthIdentities(filters?, config?, context?): Promise<AuthIdentityDTO[]>

  // Provider identity management
  createProviderIdentities(data, context?): Promise<ProviderIdentityDTO[]>
  updateProviderIdentities(data, context?): Promise<ProviderIdentityDTO[]>
  deleteProviderIdentities(ids, context?): Promise<void>

  // MFA
  createMfaFactor(authIdentityId, provider, context?): Promise<MfaFactorDTO>
  verifyMfaFactor(authIdentityId, provider, code, context?): Promise<boolean>
  deleteMfaFactor(id, context?): Promise<void>
  generateMfaRecoveryCodes(authIdentityId, count, context?): Promise<string[]>
  verifyMfaRecoveryCode(authIdentityId, code, context?): Promise<boolean>
}
```

### 4.2 AuthScope

`AuthScope` is an enum/string passed to authentication calls to distinguish admin from store context:
- `AuthScope.ADMIN` — validates against admin user actor type
- `AuthScope.STORE` — validates against customer actor type

Providers may enforce different rules per scope (e.g., the `emailpass` provider looks up different actor types depending on the scope).

### 4.3 Authentication Response

```typescript
type AuthenticationResponse = {
  success: boolean
  authIdentity?: AuthIdentityDTO
  error?: string
  status?: number         // HTTP error code
  location?: string       // OAuth redirect URL
  requires_mfa?: boolean  // MFA challenge required
}
```

### 4.4 Provider Resolution

Providers are resolved from the MedusaJS container by the key `auth_<provider_id>`. The module service delegates the actual credential validation to the resolved provider without coupling to any specific implementation.

---

## 5. Security Design

### 5.1 Password Storage

The `emailpass` provider stores passwords as bcrypt hashes in `provider_metadata`. The hash is generated with a unique salt per credential. The service layer never exposes `provider_metadata` in API responses.

### 5.2 Token Lifecycle

JWTs issued by the framework contain:
- `actor_id` — the resolved actor (user or customer) ID
- `actor_type` — `"user"` or `"customer"`
- `auth_identity_id` — the `AuthIdentity` ID
- Standard JWT claims (`iat`, `exp`)

Token refresh is handled by the HTTP framework's `/admin/auth/token/refresh` endpoint, not the module.

### 5.3 MFA Security Properties

- Recovery codes are hashed with bcrypt before storage
- Each code is single-use (soft-deleted on consumption)
- At most one active MFA factor per (identity, provider) — enforced by partial unique index
- `pending` factors are not used for authentication until verified

---

## 6. Provider Extension Pattern

Custom providers extend `AbstractAuthModuleProvider` from `@medusajs/framework/utils`:

```typescript
class CustomAuthProvider extends AbstractAuthModuleProvider {
  static PROVIDER = "custom"
  static DISPLAY_NAME = "Custom SSO"

  async authenticate(data, authIdentityProviderService) {
    // validate credentials, upsert ProviderIdentity via authIdentityProviderService
    return { success: true, authIdentity }
  }

  async validateCallback(data, authIdentityProviderService) {
    // handle OAuth callback
    return { success: true, authIdentity }
  }
}
```

---

## 7. Known Constraints

- `ProviderIdentity` has no `deleted_at` — it uses a physical-delete pattern for cleanup
- `app_metadata` is a freeform JSON field; schema validation is the responsibility of the caller
- MFA feature requires the auth module providers to implement `IMFAProvider` in addition to the base authentication interface
