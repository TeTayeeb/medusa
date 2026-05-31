# SpecKit — Auth Module

**Module:** `@medusajs/auth`  
**Version:** Medusa v2.15.4  
**Document Type:** Functional & Technical Specification

---

## 1. Functional Requirements

### FR-1: Identity Management

| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | System SHALL maintain one `AuthIdentity` per logical user regardless of authentication provider count | MUST |
| FR-1.2 | `AuthIdentity` SHALL be agnostic of actor type (admin user or customer) | MUST |
| FR-1.3 | System SHALL store actor reference in `app_metadata` (actor_id, actor_type) | MUST |
| FR-1.4 | System SHALL support merging multiple provider identities under one auth identity | MUST |
| FR-1.5 | Deleting an auth identity SHALL cascade to all provider identities and MFA records | MUST |

### FR-2: Provider-Based Authentication

| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | System SHALL support email/password authentication via `emailpass` provider | MUST |
| FR-2.2 | System SHALL support OAuth 2.0 authentication via `github` and `google` providers | MUST |
| FR-2.3 | System SHALL support custom auth providers via `AbstractAuthModuleProvider` interface | MUST |
| FR-2.4 | Provider routing SHALL be based on the `provider` string parameter | MUST |
| FR-2.5 | OAuth providers SHALL return a redirect `location` URL for the authorization flow | MUST |
| FR-2.6 | System SHALL differentiate between admin and store authentication scopes | MUST |
| FR-2.7 | `entity_id` + `provider` combination SHALL be globally unique | MUST |

### FR-3: Credential Security

| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | Passwords SHALL be stored as bcrypt hashes; plaintext SHALL never be persisted | MUST |
| FR-3.2 | `provider_metadata` SHALL never be returned in any API response | MUST |
| FR-3.3 | JWT issuance SHALL be the responsibility of the HTTP framework, not the module | MUST |
| FR-3.4 | `user_metadata` SHALL be updatable by the authenticated user | SHOULD |

### FR-4: Multi-Factor Authentication

| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | System SHALL support enrolling MFA factors per auth identity | MUST |
| FR-4.2 | MFA factor status SHALL progress: `pending` → `enabled` → soft-deleted | MUST |
| FR-4.3 | At most one active MFA factor per (identity, provider) combination SHALL be allowed | MUST |
| FR-4.4 | System SHALL require MFA verification before JWT issuance when an enabled factor exists | MUST |
| FR-4.5 | System SHALL generate single-use recovery codes for MFA bypass | MUST |
| FR-4.6 | Recovery codes SHALL be stored as bcrypt hashes; plaintext returned only at generation time | MUST |
| FR-4.7 | A consumed recovery code SHALL be immediately invalidated (soft-deleted) | MUST |

---

## 2. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Password hash verification time (bcrypt) | 100–300ms per verification (cost factor 10) |
| NFR-2 | OAuth callback processing | < 2s including external token exchange |
| NFR-3 | `provider_metadata` exposure prevention | Zero leakage; enforced at DTO serialization layer |
| NFR-4 | MFA uniqueness enforcement | Partial unique index at DB level |
| NFR-5 | Authentication failure response | Consistent error message to prevent user enumeration |

---

## 3. API Specification

### POST /store/auth/emailpass — Authenticate with Email/Password

**Request Body:**
```json
{
  "email": "customer@example.com",
  "password": "secure_password_123"
}
```

**Success Response:** `200 OK`
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**MFA Required Response:** `200 OK`
```json
{
  "requires_mfa": true,
  "auth_identity_id": "authid_01",
  "mfa_providers": ["totp"]
}
```

**Error Response:** `401 Unauthorized`
```json
{
  "type": "unauthorized",
  "message": "Invalid credentials"
}
```

### GET /store/auth/github — Initiate GitHub OAuth

**Response:** `302 Found`
```
Location: https://github.com/login/oauth/authorize?client_id=...&state=...
```

### GET /store/auth/github/callback — OAuth Callback

**Query Parameters:** `code`, `state`  
**Success Response:** `200 OK` with JWT token

### DELETE /admin/auth — Logout

**Response:** `200 OK`
```json
{ "success": true }
```

---

## 4. Provider Interface Specification

```typescript
abstract class AbstractAuthModuleProvider {
  // Provider metadata
  static PROVIDER: string       // unique provider ID
  static DISPLAY_NAME: string   // human-readable name

  // Direct authentication (email/password style)
  abstract authenticate(
    data: Record<string, unknown>,
    authIdentityProviderService: AuthIdentityProviderService
  ): Promise<AuthenticationResponse>

  // OAuth callback verification
  abstract validateCallback(
    data: Record<string, unknown>,
    authIdentityProviderService: AuthIdentityProviderService
  ): Promise<AuthenticationResponse>
}

type AuthenticationResponse = {
  success: boolean
  authIdentity?: AuthIdentityDTO
  error?: string
  status?: number      // HTTP status code for error responses
  location?: string    // OAuth redirect URL
  requires_mfa?: boolean
}
```

---

## 5. Data Validation Rules

| Field | Rule |
|---|---|
| `ProviderIdentity.entity_id` | Non-empty; unique per provider |
| `ProviderIdentity.provider` | Non-empty; must match a registered provider ID |
| `AuthMfaFactor.status` | Must be `pending` or `enabled` (soft-delete for removal) |
| `AuthMfaRecoveryCode.code_hash` | bcrypt hash; unique per auth identity |
| `AuthIdentity.app_metadata` | Valid JSON; `actor_id` and `actor_type` recommended fields |

---

## 6. Error Conditions

| Error | Type | HTTP Status |
|---|---|---|
| Invalid credentials | `unauthorized` | 401 |
| Provider not found/registered | `NOT_FOUND` | 404 |
| Auth identity not found | `NOT_FOUND` | 404 |
| OAuth state mismatch | `unauthorized` | 401 |
| MFA code invalid or expired | `unauthorized` | 401 |
| Recovery code already used | `unauthorized` | 401 |
| Duplicate active MFA factor | `INVALID_DATA` | 422 |

---

## 7. Security Specifications

### Anti-Enumeration
Authentication failure responses for invalid email and invalid password SHOULD return the same error message to prevent username enumeration attacks.

### Session Invalidation
Logout (`DELETE /admin/auth`) revokes the current JWT. The HTTP framework handles token blacklisting or short-TTL token strategies.

### OAuth State Parameter
The OAuth state parameter MUST be validated on callback to prevent CSRF attacks. The `github` and `google` providers verify the state value against the session.

### Recovery Code Constraints
- Recovery codes SHALL be generated in sets (e.g., 10 codes per batch)
- Each code SHALL be single-use — consumption soft-deletes the record immediately
- Old recovery code sets SHOULD be invalidated when new codes are generated
