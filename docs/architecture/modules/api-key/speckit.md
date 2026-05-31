# SpecKit — API Key Module

**Module:** `@medusajs/api-key`  
**Version:** Medusa v2.15.4  
**Document Type:** Functional & Technical Specification

---

## 1. Functional Requirements

### FR-1: Key Creation

| ID | Requirement | Priority |
|---|---|---|
| FR-1.1 | System SHALL support creating secret keys for programmatic admin access | MUST |
| FR-1.2 | System SHALL support creating publishable keys for storefront context identification | MUST |
| FR-1.3 | Secret key tokens SHALL be stored as bcrypt hashes; plaintext returned only at creation | MUST |
| FR-1.4 | System SHALL generate a redacted token representation for display in admin UI | MUST |
| FR-1.5 | Keys SHALL record the creating admin user ID in `created_by` | MUST |
| FR-1.6 | Keys SHALL have a human-readable `title` field | MUST |

### FR-2: Key Authentication

| ID | Requirement | Priority |
|---|---|---|
| FR-2.1 | System SHALL validate secret key tokens via bcrypt comparison | MUST |
| FR-2.2 | System SHALL immediately reject revoked keys (`revoked_at IS NOT NULL`) | MUST |
| FR-2.3 | System SHALL immediately reject deleted keys (`deleted_at IS NOT NULL`) | MUST |
| FR-2.4 | System SHALL update `last_used_at` on successful secret key authentication | MUST |
| FR-2.5 | Publishable keys SHALL be resolvable by token value without authentication | MUST |

### FR-3: Key Revocation

| ID | Requirement | Priority |
|---|---|---|
| FR-3.1 | System SHALL support soft-revoking keys by setting `revoked_at` and `revoked_by` | MUST |
| FR-3.2 | Revocation SHALL take effect immediately for all subsequent auth requests | MUST |
| FR-3.3 | Revoked keys SHALL remain in the database for audit purposes | MUST |
| FR-3.4 | System SHALL prevent revoking an already-revoked key | SHOULD |

### FR-4: Key Management

| ID | Requirement | Priority |
|---|---|---|
| FR-4.1 | System SHALL support updating key `title` | MUST |
| FR-4.2 | Token value, type, and `created_by` SHALL be immutable after creation | MUST |
| FR-4.3 | System SHALL support hard-deleting keys (permanent removal) | MUST |
| FR-4.4 | System SHALL support listing keys with filter by `type` | MUST |
| FR-4.5 | System SHALL support filtering active vs. revoked keys | MUST |

### FR-5: Sales Channel Association (Publishable Keys)

| ID | Requirement | Priority |
|---|---|---|
| FR-5.1 | Publishable keys SHALL be associable with one or more sales channels | MUST |
| FR-5.2 | Store API requests carrying a publishable key SHALL be scoped to its linked sales channels | MUST |
| FR-5.3 | System SHALL support adding and removing sales channel associations | MUST |
| FR-5.4 | Deleting a publishable key SHALL clean up sales channel link records | MUST |

---

## 2. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Secret key authentication latency | < 200ms (includes bcrypt comparison ~100ms) |
| NFR-2 | Revocation propagation time | Immediate (synchronous DB update; no cache delay) |
| NFR-3 | Token uniqueness | Enforced by unique index on `token` column |
| NFR-4 | Redacted format | Must preserve type prefix and last 4 chars: `sk_...xxxx` |
| NFR-5 | Audit trail completeness | `created_by`, `revoked_by`, `revoked_at`, `last_used_at` always recorded |

---

## 3. API Specification

### POST /admin/api-keys — Create API Key

**Request Body:**
```json
{
  "title": "Production Backend Key",
  "type": "secret"
}
```

**Response:** `201 Created`
```json
{
  "api_key": {
    "id": "apk_01HXXXX",
    "title": "Production Backend Key",
    "type": "secret",
    "redacted": "sk_...a3f8",
    "token": "sk_live_abc123...xyz789",
    "created_by": "user_01",
    "last_used_at": null,
    "revoked_at": null,
    "created_at": "2025-01-01T00:00:00Z"
  }
}
```

> ⚠️ **`token` is only present at creation time.** It cannot be retrieved again.

### GET /admin/api-keys — List API Keys

**Query Parameters:**
- `type`: `secret` | `publishable`
- `q`: search by `title` or `redacted`
- `limit`, `offset`: pagination

**Response:** `200 OK`
```json
{
  "api_keys": [
    {
      "id": "apk_01HXXXX",
      "title": "Production Backend Key",
      "type": "secret",
      "redacted": "sk_...a3f8",
      "last_used_at": "2025-06-15T12:30:00Z",
      "revoked_at": null
    }
  ],
  "count": 1,
  "offset": 0,
  "limit": 20
}
```

### POST /admin/api-keys/:id/revoke — Revoke Key

**Response:** `200 OK`
```json
{
  "api_key": {
    "id": "apk_01HXXXX",
    "revoked_at": "2025-06-20T09:00:00Z",
    "revoked_by": "user_01"
  }
}
```

### POST /admin/api-keys/:id/sales-channels/batch — Link Sales Channels

**Request Body:**
```json
{
  "add": ["sc_01", "sc_02"],
  "remove": ["sc_03"]
}
```

---

## 4. Workflow Specifications

### createApiKeysWorkflow

**Input:** `{ title: string, type: "secret" | "publishable", created_by: string }`

**Steps:**
1. Generate random token (prefixed: `sk_` or `pk_`)
2. For `secret`: compute bcrypt hash and salt; store hash
3. For `publishable`: store plaintext token
4. Compute `redacted` string
5. Persist `ApiKey` record
6. Return `ApiKeyDTO` with `token` field (plaintext) included only in workflow output

**Compensation:** `deleteApiKeysStep` — hard-delete the created record

### revokeApiKeysWorkflow

**Input:** `{ ids: string[], revoked_by: string }`

**Steps:**
1. Verify keys exist and are not already revoked
2. Set `revoked_at = NOW()`, `revoked_by = input.revoked_by`

**Compensation:** Unrevoking is not supported (revocation is intentional)

---

## 5. Data Validation Rules

| Field | Rule |
|---|---|
| `ApiKey.title` | Non-empty string; max 255 chars |
| `ApiKey.type` | Must be `secret` or `publishable` |
| `ApiKey.created_by` | Required; must be a valid admin user ID |
| Token format | Prefix must match type: `sk_` for secret, `pk_` for publishable |

---

## 6. Error Conditions

| Error | Type | HTTP Status |
|---|---|---|
| API key not found | `NOT_FOUND` | 404 |
| Cannot update revoked key | `NOT_ALLOWED` | 400 |
| Key is already revoked | `NOT_ALLOWED` | 400 |
| Invalid API key during authentication | `unauthorized` | 401 |
| Revoked key used for authentication | `unauthorized` | 401 |
| Cannot associate sales channels with secret key | `INVALID_DATA` | 422 |

---

## 7. Security Specifications

### Token Prefix Convention
| Type | Prefix | Example |
|---|---|---|
| Secret key | `sk_` | `sk_live_abc123...` |
| Publishable key | `pk_` | `pk_live_xyz789...` |

Prefixes allow type identification at a glance and enable infrastructure-level filtering (e.g., detecting secret keys accidentally committed to source control).

### Rotation Protocol
1. Create new key → receive plaintext token
2. Update all consumers with new token
3. Verify new token works in production
4. Revoke old token — takes effect immediately

A **zero-downtime rotation** requires a brief overlap window between new key creation and old key revocation.
