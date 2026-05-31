# Software Design Document — API Key Module

**Module:** `@medusajs/api-key`  
**Version:** Medusa v2.15.4  
**Status:** Production  
**Last Updated:** 2025

---

## 1. Purpose and Scope

This document describes the internal design, data model, key lifecycle, and security architecture of the API Key Module. The module manages two classes of machine credentials: secret keys for backend API access and publishable keys for storefront context identification.

---

## 2. System Context

The API Key Module is consumed in two distinct contexts:

1. **Admin authentication middleware** — validates `Authorization: Bearer <token>` headers for programmatic admin API access using secret keys
2. **Storefront context resolution** — resolves `x-publishable-api-key` headers to sales channel scope using publishable keys

```
Admin API Request → Auth Middleware → API Key Module (verify secret key hash)
                                   → Resolve actor from key metadata

Store API Request → Channel Middleware → API Key Module (lookup publishable key)
                                       → Resolve associated SalesChannels via Module Link
```

---

## 3. Data Model

### 3.1 Entity Summary

There is a single entity in this module:

| Entity | Table | PK Prefix |
|---|---|---|
| `ApiKey` | `api_key` | `apk_` |

### 3.2 ApiKey Fields

```
ApiKey {
  id           : string (PK, prefix: apk)
  token        : string (unique)       -- bcrypt hash (secret) or plaintext (publishable)
  salt         : string                -- bcrypt salt
  redacted     : string (searchable)   -- e.g. "sk_...abcd"
  title        : string (searchable)   -- human-readable label
  type         : ApiKeyType (secret | publishable)
  last_used_at : datetime | null       -- updated on each authenticated request
  created_by   : string                -- admin user ID
  revoked_by   : string | null         -- admin user ID who revoked, if applicable
  revoked_at   : datetime | null       -- null = active, set = revoked
  created_at   : datetime
  updated_at   : datetime
  deleted_at   : datetime | null
}
```

**Indexes:**
- `IDX_api_key_token` — unique on `token`; used for O(1) token lookup during authentication
- `IDX_api_key_revoked_at` — supports efficient filtering of active vs. revoked keys
- `IDX_api_key_redacted` — supports search by redacted value in admin UI
- `IDX_api_key_type` — supports filtering keys by type

**Note:** There is no `status` enum field. A key is considered active when `revoked_at IS NULL AND deleted_at IS NULL`. This keeps the query simple and index-efficient.

### 3.3 Active Key Query

```sql
SELECT * FROM api_key
WHERE revoked_at IS NULL
  AND deleted_at IS NULL
  AND type = 'secret';
```

---

## 4. Service Layer

### 4.1 IApiKeyModuleService

```typescript
interface IApiKeyModuleService {
  createApiKeys(data: CreateApiKeyDTO[], context?): Promise<ApiKeyDTO[]>
  updateApiKeys(data: UpdateApiKeyDTO[], context?): Promise<ApiKeyDTO[]>
  deleteApiKeys(ids: string[], context?): Promise<void>
  revokeApiKeys(ids: string[], context?): Promise<ApiKeyDTO[]>
  
  retrieveApiKey(id: string, config?, context?): Promise<ApiKeyDTO>
  listApiKeys(filters?: FilterableApiKeyProps, config?, context?): Promise<ApiKeyDTO[]>
  listAndCountApiKeys(filters?, config?, context?): Promise<[ApiKeyDTO[], number]>
  
  // Authentication
  authenticate(token: string, context?): Promise<ApiKeyDTO>
  
  // Soft-delete
  softDeleteApiKeys(ids: string[], config?, context?): Promise<void>
  restoreApiKeys(ids: string[], config?, context?): Promise<void>
}
```

### 4.2 Key Creation Algorithm

On `createApiKeys`:

1. For **secret keys**:
   a. Generate a cryptographically secure random token string (e.g., `sk_` prefix + 32 random bytes as hex)
   b. Generate a bcrypt salt
   c. Compute `hash = bcrypt(token, salt)`
   d. Compute `redacted = token.slice(0, 6) + "..." + token.slice(-4)`
   e. Persist `{ token: hash, salt, redacted, ... }`
   f. Return DTO with the **plaintext token** — this is the only time it is available

2. For **publishable keys**:
   a. Generate a random token string (e.g., `pk_` prefix)
   b. Store plaintext token directly (no hashing required)
   c. Compute and store `redacted` value
   d. Return DTO with plaintext token

### 4.3 Authentication (Secret Key Verification)

On `authenticate(token)`:

1. Retrieve the `ApiKey` record where `token` index lookup returns the candidate (index is on the hash for secret keys — so this requires a scan with bcrypt comparison, or a pre-hashed lookup depending on implementation variant)
2. Verify `revoked_at IS NULL` and `deleted_at IS NULL`
3. Update `last_used_at = now()` on successful verification
4. Return the `ApiKeyDTO` on success

**Implementation note:** Because bcrypt hashes are non-reversible and non-comparable without the original token, the module uses a two-phase approach: first look up candidate keys by any available discriminator, then verify the bcrypt hash. For performance, keys may also store a fast-lookup token prefix indexed separately.

### 4.4 Revocation

On `revokeApiKeys(ids)`:

```sql
UPDATE api_key
SET revoked_at = NOW(), revoked_by = <caller_id>
WHERE id IN (:ids) AND revoked_at IS NULL;
```

Revocation is a soft operation. The record is preserved for audit purposes. The key immediately becomes invalid for authentication since all auth checks verify `revoked_at IS NULL`.

---

## 5. ApiKeyDTO (Returned Fields)

The service DTO **never includes** the raw `token` (hash) or `salt` fields:

```typescript
type ApiKeyDTO = {
  id: string
  title: string
  type: "secret" | "publishable"
  redacted: string
  last_used_at: Date | null
  created_by: string
  revoked_by: string | null
  revoked_at: Date | null
  created_at: Date
  updated_at: Date
  // token is ONLY returned at creation time in a separate field
}

type CreatedApiKeyDTO = ApiKeyDTO & {
  token: string  // plaintext — only available at creation time
}
```

---

## 6. Module Link: Publishable Key ↔ Sales Channel

Publishable keys are associated with sales channels via a module link defined in `@medusajs/link-modules`:

```typescript
defineLink(
  ApiKeyModule.linkable.apiKey,
  SalesChannelModule.linkable.salesChannel
)
```

The `linkSalesChannelsToPublishableKeyWorkflow` manages this association (add/remove/replace patterns). The storefront context middleware resolves the publishable key from the request header and fetches the linked sales channels via `remoteQuery`.

---

## 7. Security Architecture

| Concern | Implementation |
|---|---|
| Secret key confidentiality | bcrypt hash stored; plaintext never persisted |
| Revocation immediacy | `revoked_at` checked on every auth request |
| Audit trail | `created_by`, `revoked_by`, `revoked_at`, `last_used_at` all persisted |
| Publishable key safety | No auth privileges; safe for client-side use |
| Unique token guarantee | Unique index on `token` column |

---

## 8. Known Constraints

- Key `title` and `type` are the only mutable fields after creation — token is immutable
- Hard delete (`deleteApiKeys`) permanently removes the record and all audit history
- `last_used_at` updates may be batched or eventually consistent in high-throughput scenarios
- Sales channel links for publishable keys are managed separately via module links, not the ApiKey service directly
