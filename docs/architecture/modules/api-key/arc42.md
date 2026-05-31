# arc42 Architecture Documentation — API Key Module

**Module:** `@medusajs/api-key`  
**Version:** Medusa v2.15.4  
**Template:** arc42 v8.2

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The API Key Module must:
- Provide secure secret key generation with hashed (non-reversible) storage
- Support publishable keys that safely identify storefront context
- Enable key revocation without losing audit history
- Support sales channel association for publishable keys via module links
- Track last-used timestamps for operational visibility
- Support key rotation workflows (revoke old → create new)

### 1.2 Quality Goals

| Quality Attribute | Priority | Scenario |
|---|---|---|
| Security | Critical | Secret key tokens must never be recoverable from the database |
| Operational Safety | High | Revoked keys must be immediately rejected with no grace period |
| Auditability | High | All key lifecycle events (creation, revocation) must be attributed to a user |
| Simplicity | Medium | Single entity model covers both key types via type discriminator |

---

## 2. Architecture Constraints

- Secret key tokens are bcrypt-hashed at creation; plaintext is never persisted
- Only `title` is mutable post-creation; token, type, created_by are immutable
- The module has no knowledge of sales channels directly — that association is via an external module link
- Hard delete is available but removes all audit history; revocation (soft operation) is preferred

---

## 3. System Scope and Context

```
┌────────────────────────────────────────────────────────────────────┐
│                      Medusa Application                            │
│                                                                    │
│  ┌──────────────────┐  verify secret key  ┌─────────────────────┐ │
│  │  Admin API Auth  │ ───────────────────► API Key Module       │ │
│  │  Middleware      │ ◄─────────────────── ApiKeyDTO | null      │ │
│  └──────────────────┘                     │                     │ │
│                                           │  ┌───────────────┐  │ │
│  ┌──────────────────┐ lookup pub key      │  │  PostgreSQL   │  │ │
│  │  Store API       │ ───────────────────►│  └───────────────┘  │ │
│  │  Channel Scope   │                     │                     │ │
│  │  Middleware      │ SalesChannel IDs    │  ┌───────────────┐  │ │
│  └──────────────────┘ ◄───────────────────── Module Link     │  │ │
│                                           │  (ApiKey↔Channel)│  │ │
│                                           └──────────────────┘  │ │
│  ┌──────────────────┐                                           │ │
│  │  Admin API       │  CRUD operations                         │ │
│  │  /admin/api-keys │ ─────────────────────────────────────────► │
│  └──────────────────┘                                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. Solution Strategy

The module uses a **single-entity, type-discriminated design**: one `ApiKey` table handles both secret and publishable keys. The `type` field controls which security properties apply (hashing for secret, plaintext for publishable).

For authentication validation, the module uses **bcrypt hash comparison**: the stored `token` field contains the hash for secret keys. Validation requires computing `bcrypt.compare(inputToken, storedHash)`.

For revocation, the module uses **soft-invalidation via timestamp**: setting `revoked_at` immediately invalidates the key for all authentication checks without deleting the audit record.

---

## 5. Building Blocks

### Level 1: Module Structure

```
@medusajs/api-key
├── services/
│   └── api-key-module-service.ts   # IApiKeyModuleService implementation
├── models/
│   └── api-key.ts
└── migrations/
```

### Level 2: Workflows

| Workflow | Package | Steps |
|---|---|---|
| `createApiKeysWorkflow` | `@medusajs/core-flows` | generateToken → hashToken → persistKey → returnPlaintext |
| `revokeApiKeysWorkflow` | `@medusajs/core-flows` | setRevokedAt → setRevokedBy |
| `deleteApiKeysWorkflow` | `@medusajs/core-flows` | hardDelete |
| `linkSalesChannelsToPublishableKeyWorkflow` | `@medusajs/core-flows` | upsert module link records |

---

## 6. Runtime View

### Scenario: Secret Key Authentication

```
1. Request arrives: POST /admin/products
   Authorization: Bearer sk_abc123xyz...
2. Admin auth middleware extracts bearer token
3. Calls apiKeyModule.authenticate("sk_abc123xyz...")
4. Module queries active secret keys (revoked_at IS NULL, type = 'secret')
5. For matching candidates: bcrypt.compare(inputToken, key.token)
6. Match found: update key.last_used_at = NOW()
7. Return ApiKeyDTO to middleware
8. Middleware resolves actor_id from key metadata, continues request
```

### Scenario: Key Rotation

```
1. Admin decides to rotate an existing secret key "sk_old..."
2. POST /admin/api-keys { title: "Production Key v2", type: "secret" }
3. New key created; plaintext returned ONCE: "sk_new..."
4. Admin updates consumer (CI/CD, service config) with new key
5. POST /admin/api-keys/:old_id/revoke
6. Old key: revoked_at = NOW(), revoked_by = admin_user_id
7. Old key immediately rejected for all subsequent requests
```

### Scenario: Storefront Channel Scoping

```
1. Store request arrives: GET /store/products
   x-publishable-api-key: pk_live_abc...
2. Channel scope middleware calls apiKeyModule.listApiKeys({ token: "pk_live_abc..." })
3. Key resolved: type = 'publishable', revoked_at IS NULL
4. Middleware uses remoteQuery to fetch linked SalesChannels via module link
5. Request context scoped to those sales channels for product/inventory queries
```

---

## 7. Deployment View

The API Key Module is a lightweight module running in the Medusa Node.js process. The bcrypt operations for authentication may add ~50-100ms per request when validating secret keys — this is by design (bcrypt cost factor provides brute-force resistance). For high-throughput admin API usage, secret key authentication results should be cached in an in-memory or Redis cache keyed by a fast token prefix lookup.

---

## 8. Crosscutting Concepts

### One-Time Plaintext Return
The `createApiKeysWorkflow` returns the plaintext token exactly once in the workflow output. This is a UX convention enforced at the workflow layer — the module stores only the hash. The admin UI must display this token prominently with a "copy" action and a warning that it cannot be retrieved again.

### Redacted Display
The `redacted` field (e.g., `sk_...xxxx`) is computed at creation time and stored for safe display in listings and audit logs without exposing the sensitive token value.

### Sales Channel Link Lifecycle
When a publishable key is deleted, the module link records (ApiKey ↔ SalesChannel) must also be cleaned up. This is handled by the `deleteApiKeysWorkflow` which coordinates link cleanup via `remoteLink.dismiss`.

---

## 9. Architecture Decisions

### ADR-1: Single Entity, Two Types
Rather than having separate `SecretKey` and `PublishableKey` entities, a single `ApiKey` entity with a `type` discriminator is used. This simplifies the data model and the admin UI, at the cost of some nullable fields that only apply to one type.

### ADR-2: Revocation Over Deletion for Audit
`revokeApiKeys` is the preferred deactivation method because it preserves the `created_by`, `revoked_by`, and timing information. This audit trail is valuable for security incident investigations.

### ADR-3: No Token Caching in Module
The module does not cache authentication results. Caching is left to the application layer (HTTP framework middleware) to allow flexible cache invalidation strategies without coupling the module to a specific cache implementation.

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Token secrecy | bcrypt hash stored; unique index on token hash |
| Immediate revocation | `revoked_at IS NULL` checked on every authenticate() call |
| Audit trail | `created_by`, `revoked_by`, `revoked_at`, `last_used_at` all persisted |
| Rotation support | Revoke + create new; both records preserved |
