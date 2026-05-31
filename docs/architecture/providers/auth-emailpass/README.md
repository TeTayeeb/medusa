# @medusajs/auth-emailpass

Email/password authentication provider for Medusa v2. Implements `IAuthProvider` via `AbstractAuthModuleProvider` and registers under `Modules.AUTH` with identifier `emailpass`.

Passwords are hashed using **scrypt** (via `scrypt-kdf`) — not bcrypt — before storage, making it resistant to GPU-accelerated brute-force attacks.

## Installation

```bash
npm install @medusajs/auth-emailpass
```

## Configuration

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/auth",
      options: {
        providers: [
          {
            resolve: "@medusajs/auth-emailpass",
            id: "emailpass",
            options: {
              // Optional: tune scrypt parameters (defaults shown)
              hashConfig: {
                logN: 15,  // CPU/memory cost (2^logN iterations)
                r: 8,      // Block size
                p: 1,      // Parallelisation factor
              },
            },
          },
        ],
      },
    },
  ],
})
```

### Options reference

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `hashConfig.logN` | `number` | — | `15` | scrypt cost factor — higher = slower, safer |
| `hashConfig.r` | `number` | — | `8` | scrypt block size |
| `hashConfig.p` | `number` | — | `1` | scrypt parallelisation parameter |

## Provider API

### `register(userData, authIdentityService)`

Creates a new `ProviderIdentity` entry with the scrypt-hashed password stored in `provider_metadata.password`.

- If an identity already exists and has no `app_metadata` (i.e. unclaimed), the existing identity is updated with the new password hash.
- If an identity exists and already has `app_metadata`, registration is rejected with `"Identity with email already exists"`.
- The password hash is **stripped** from the returned `authIdentity` object.

### `authenticate(userData, authIdentityService)`

1. Looks up `ProviderIdentity` by `entity_id` (email).
2. Retrieves `provider_metadata.password` (base64 scrypt hash).
3. Calls `scrypt-kdf.verify(hashBuffer, password)`.
4. On success, returns `authIdentity` with password stripped.
5. On any failure returns `{ success: false, error: "Invalid email or password" }` (no information leakage).

### `update(data, authIdentityService)`

Updates `provider_metadata.password` for an existing identity. Requires `entity_id`. If `password` is absent or non-string, the call is a no-op (returns `{ success: true }`).

## JWT issuance

This provider does **not** issue JWT tokens. After a successful `authenticate` or `register`, the Auth Module framework generates and returns a short-lived JWT to the caller. The provider only handles credential validation and identity management.

## Security notes

- Error messages are deliberately generic to prevent email enumeration.
- scrypt default parameters (`logN: 15`) target ~100 ms on typical server hardware.
- Passwords are never stored in plain text or logged.
- The `provider_metadata.password` field is deleted from every returned `authIdentity` object before leaving the service.

## Typical request flow

```
POST /auth/customer/emailpass
  Body: { email, password }
  → Auth Module → EmailPassAuthService.authenticate
  → ProviderIdentity lookup + scrypt verify
  → { success: true, authIdentity }
  → Auth Module issues JWT
  → { token }
```
