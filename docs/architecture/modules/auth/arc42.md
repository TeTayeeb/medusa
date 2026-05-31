# arc42 Architecture Documentation — Auth Module

**Module:** `@medusajs/auth`  
**Version:** Medusa v2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The Auth Module must:
- Support multiple authentication providers (email/password, OAuth 2.0) with a unified interface
- Maintain a single stable `AuthIdentity` per user regardless of how many providers they use
- Decouple authentication identity from actor entities (Customer, Admin User)
- Support multi-factor authentication (TOTP or other provider-based MFA)
- Provide secure recovery from MFA loss via single-use backup codes
- Enable new authentication providers without modifying core module code

### 1.2 Quality Goals

| Quality Attribute | Priority | Scenario |
|---|---|---|
| Security | Critical | Passwords must never be stored in plaintext; MFA codes must be single-use |
| Extensibility | Critical | New OAuth providers must be addable without core module changes |
| Identity Stability | High | Actor ID must remain stable across provider changes and re-authentication |
| Separation of Concerns | High | Auth module must not know about Customer or User domain concepts |

---

## 2. Architecture Constraints

- JWT issuance is the responsibility of the HTTP framework, not the module
- The module must not store plaintext passwords or OAuth tokens in API-accessible fields
- `ProviderIdentity` is provider-specific; one user may have multiple provider identities
- The `app_metadata` JSON field is the only channel between auth and actor resolution

---

## 3. System Scope and Context

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Medusa Application                            │
│                                                                      │
│  ┌──────────────┐   authenticate()   ┌────────────────────────────┐ │
│  │ Admin/Store  │ ──────────────────► Auth Module                 │ │
│  │ API Routes   │ ◄────────────────── AuthenticationResponse      │ │
│  └──────────────┘                    │                            │ │
│          │                           │  ┌──────────────────────┐  │ │
│          │ mint JWT                  │  │ Auth Providers       │  │ │
│          ▼                           │  │ emailpass | github   │  │ │
│  ┌──────────────┐                    │  │ google | custom...   │  │ │
│  │ HTTP Framework│                   │  └──────────────────────┘  │ │
│  │ (JWT issuer)  │                   │                            │ │
│  └──────────────┘                    │  ┌──────────────────────┐  │ │
│                                      │  │    PostgreSQL         │  │ │
│  ┌──────────────┐  module link       │  └──────────────────────┘  │ │
│  │ Customer /   │◄───────────────────┤                            │ │
│  │ User Module  │  actor_id lookup   └────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

The module uses an **Identity Aggregation pattern**: multiple `ProviderIdentity` records (one per external provider) are aggregated under a single `AuthIdentity`. The `AuthIdentity.app_metadata` field acts as the stable reference to the actor domain.

For authentication flows, the module uses the **Provider Strategy pattern**: each authentication mechanism is encapsulated in an `AbstractAuthModuleProvider` implementation. The module service acts as a router, selecting the correct provider based on the `provider` parameter.

MFA uses a **challenge-response pattern**: initial authentication with valid primary credentials triggers a challenge state if an enabled MFA factor exists. The client must complete the MFA challenge before a JWT is issued.

---

## 5. Building Blocks

### Level 1: Module Structure

```
@medusajs/auth
├── services/
│   ├── auth-module.ts          # IAuthModuleService implementation
│   ├── auth-provider.ts        # Provider resolution & delegation
│   └── mfa-provider.ts         # MFA coordination
├── models/
│   ├── auth-identity.ts
│   ├── provider-identity.ts
│   ├── auth-mfa-factor.ts
│   └── auth-mfa-recovery-code.ts
└── migrations/
```

### Level 2: Provider Packages

| Package | Provider ID | Mechanism |
|---|---|---|
| `@medusajs/auth-emailpass` | `emailpass` | Email + bcrypt password |
| `@medusajs/auth-github` | `github` | OAuth 2.0 via GitHub API |
| `@medusajs/auth-google` | `google` | OAuth 2.0 via Google API |
| Custom | any string | `AbstractAuthModuleProvider` extension |

---

## 6. Runtime View

### Scenario: Email/Password Login (Store Customer)

```
1. POST /store/auth/emailpass { email, password }
2. API route calls authModule.authenticate("emailpass", { email, password }, "store")
3. Auth module resolves emailpass provider from container
4. Provider looks up ProviderIdentity where entity_id = email AND provider = "emailpass"
5. Provider bcrypt.compare(password, providerIdentity.provider_metadata.hash)
6. If match: check for enabled MFA factors on the AuthIdentity
7a. No MFA: return { success: true, authIdentity }
7b. MFA enabled: return { success: true, requires_mfa: true, authIdentity }
8. HTTP framework reads app_metadata.actor_id, actor_type from authIdentity
9. Framework mints JWT: { actor_id, actor_type, auth_identity_id }
10. JWT returned to client in response
```

### Scenario: OAuth Flow (GitHub)

```
1. GET /store/auth/github
2. Auth module calls github provider.authenticate({}, scope)
3. Provider returns { success: false, location: "https://github.com/login/oauth/authorize?..." }
4. Framework redirects browser to GitHub
5. GitHub redirects to /store/auth/github/callback?code=abc&state=xyz
6. Auth module calls github provider.validateCallback({ code, state }, scope)
7. Provider exchanges code for GitHub access token
8. Provider fetches GitHub user profile (id, login, email)
9. Provider upserts ProviderIdentity { entity_id: githubId, provider: "github" }
10. Provider upserts AuthIdentity if first login
11. Returns { success: true, authIdentity }
12. Framework mints JWT and returns to client
```

---

## 7. Deployment View

The Auth Module and all provider packages run in the same Medusa Node.js process. OAuth providers make outbound HTTPS calls to external authorization servers (GitHub, Google APIs). Provider packages are configured and loaded at startup via `medusa-config.ts`.

---

## 8. Crosscutting Concepts

### Provider Metadata Privacy
`provider_metadata` on `ProviderIdentity` and `AuthMfaFactor` is never included in API responses. It is an internal storage field for sensitive data (password hashes, TOTP secrets, OAuth tokens).

### app_metadata as Actor Bridge
`AuthIdentity.app_metadata` is the only coupling point between the auth module and the actor domains. When a Customer registers, the `actor_id` (Customer ID) and `actor_type: "customer"` are stored in `app_metadata`. This allows the auth module to remain completely domain-agnostic.

### MFA Partial Unique Index
The partial unique index `WHERE status IN ('pending', 'enabled')` on `(auth_identity_id, provider)` enforces that only one active enrollment per MFA method per identity exists, while allowing multiple soft-deleted historical records.

---

## 9. Architecture Decisions

### ADR-1: AuthIdentity Separate from Actor
Authentication identity is separated from Customer/User to enable: multiple providers per user, sharing auth identities across actor types (future), and keeping the auth module free from domain concerns.

### ADR-2: JWT Issued by Framework, Not Module
The module returns authentication results; JWT minting is the framework's responsibility. This keeps cryptographic key management centralized and allows token format changes without module changes.

### ADR-3: MFA as Module Feature, Not Provider Feature
MFA is implemented at the module level (not within individual providers). This means any provider-authenticated identity can optionally have MFA applied as a second layer, regardless of which primary provider is used.

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Password security | bcrypt with per-credential salt; `provider_metadata` never exposed in API |
| Provider extensibility | `AbstractAuthModuleProvider` interface; provider resolved from DI container |
| MFA correctness | Partial unique index; single-use recovery codes; soft-delete on consumption |
| Identity stability | `AuthIdentity.id` is permanent and never reused |
