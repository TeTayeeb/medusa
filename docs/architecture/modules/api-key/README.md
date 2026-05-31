# API Key Module

**Package:** `@medusajs/api-key`  
**Version:** Medusa v2.15.4  
**Module Key:** `Modules.API_KEY`

## Overview

The API Key Module manages two distinct categories of API keys in Medusa v2: **secret keys** for programmatic backend access and **publishable keys** for client-side storefront identification. It handles secure key generation, hashed storage, revocation, and rotation — providing a clean administrative interface for managing long-lived machine credentials.

## Key Concepts

### ApiKey Entity

There is a single `ApiKey` entity that represents both key types, discriminated by the `type` field:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Primary key with prefix `apk` |
| `token` | `string` | The actual key value (stored as bcrypt hash for secret keys) |
| `salt` | `string` | Bcrypt salt used during token generation |
| `redacted` | `string` | Partially masked token for display (e.g., `sk_...xxxx`) |
| `title` | `string` | Human-readable label for the key |
| `type` | `enum` | `secret` or `publishable` |
| `last_used_at` | `datetime\|null` | Timestamp of the last successful authenticated request |
| `created_by` | `string` | Admin user ID who created the key |
| `revoked_by` | `string\|null` | Admin user ID who revoked the key |
| `revoked_at` | `datetime\|null` | Revocation timestamp; `null` means the key is active |

A key is considered **active** when `revoked_at IS NULL`. All indexes on `revoked_at` support efficient filtering of active vs. revoked keys.

### Secret Keys

Secret keys (`type: "secret"`) are used for programmatic access to the Medusa Admin API (server-to-server integrations, CI/CD pipelines, backend services). Key properties:

- **Hashed storage:** The raw token is never stored. On creation, the module generates a cryptographically secure random token, hashes it with bcrypt, and stores the hash in `token` along with the `salt`. Only the plaintext token is returned to the caller at creation time — it cannot be retrieved later.
- **Authentication:** Incoming `Authorization: Bearer <token>` headers are validated by comparing the provided token against the stored hash.
- **Redacted display:** The `redacted` field stores a masked version (showing only the first few and last few characters) for safe display in the admin UI.
- **Rotation:** Rotating a secret key requires revoking the old key and creating a new one. The `revokeApiKeysWorkflow` soft-revokes by setting `revoked_at` and `revoked_by`.

### Publishable Keys

Publishable keys (`type: "publishable"`) serve a different purpose: they identify the **storefront context** for API requests. When a storefront sends a `x-publishable-api-key` header, Medusa uses it to scope the request to a specific set of sales channels.

Key properties:
- **Plaintext storage:** Publishable keys do not require hashing since they are not authentication credentials — they are not secret.
- **Sales channel scoping:** Via a module link (`PublishableApiKeySalesChannel`), a publishable key is associated with one or more sales channels. Any API request carrying the key is automatically scoped to those channels for product availability, pricing, and inventory queries.
- **Frontend safe:** Since they carry no authentication privileges, publishable keys can safely be included in client-side JavaScript bundles.

### Key Rotation Pattern

The recommended rotation workflow:
1. Create a new key via `createApiKeysWorkflow`
2. Deploy the new key to the consumer
3. Revoke the old key via `revokeApiKeysWorkflow`

There is no in-place update of the token value; rotation always creates a new key record.

## Key Workflows

| Workflow | Description |
|---|---|
| `createApiKeysWorkflow` | Generates a new key, hashes it (secret), returns plaintext token once |
| `updateApiKeysWorkflow` | Updates metadata (title only; token is immutable) |
| `deleteApiKeysWorkflow` | Hard-deletes a key record |
| `revokeApiKeysWorkflow` | Soft-revokes by setting `revoked_at` / `revoked_by` |
| `linkSalesChannelsToPublishableKeyWorkflow` | Associates sales channels with a publishable key |

## Admin API

| Endpoint | Description |
|---|---|
| `GET /admin/api-keys` | List all API keys (paginated, filterable by `type`) |
| `POST /admin/api-keys` | Create a new API key (returns plaintext token once) |
| `GET /admin/api-keys/:id` | Get API key details (redacted token only) |
| `POST /admin/api-keys/:id` | Update key title |
| `DELETE /admin/api-keys/:id` | Delete an API key |
| `POST /admin/api-keys/:id/revoke` | Revoke an API key |
| `POST /admin/api-keys/:id/sales-channels/batch` | Link/unlink sales channels to a publishable key |

## Module Links

- **ApiKey ↔ SalesChannel** — links publishable keys to one or more sales channels (defined in `@medusajs/link-modules`)
- The `x-publishable-api-key` header on storefront requests is resolved to the associated sales channels via this link

## Security Considerations

1. **Secret keys are one-time visible** — the plaintext token is returned only at creation time. If lost, the key must be rotated.
2. **Bcrypt hashing** — secret key tokens are hashed with bcrypt before persistence, providing resistance against database compromise.
3. **Revocation over deletion** — revoking (setting `revoked_at`) preserves audit history. Hard deletion removes the record entirely.
4. **`last_used_at` tracking** — allows administrators to identify stale keys and revoke them proactively.
5. **Publishable keys carry no auth privileges** — they only provide sales channel scoping and cannot be used to authenticate admin operations.

## Configuration

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    { resolve: "@medusajs/api-key" }
  ]
})
```
