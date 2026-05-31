# Software Design Document — @medusajs/auth-emailpass

## 1. Purpose

Provide credential-based (email + password) authentication for Medusa actors (customers, admins). The provider handles password hashing, storage, and verification; JWT issuance is delegated to the Auth Module framework.

## 2. Architecture

```
Modules.AUTH
  └── ModuleProvider (auth-emailpass)
        └── EmailPassAuthService (AbstractAuthModuleProvider)
              ├── register(userData, authIdentityService)
              ├── authenticate(userData, authIdentityService)
              ├── update(data, authIdentityService)
              ├── hashPassword(password) → base64 scrypt hash
              └── upsertAuthIdentity(type, { email, password, authIdentityService })
```

`authIdentityService` is provided by the Auth Module and wraps the `ProviderIdentity` repository. The provider never accesses the database directly.

## 3. Data model

```
ProviderIdentity {
  entity_id:         string  // email address (unique per provider)
  provider:          string  // "emailpass"
  provider_metadata: {
    password:        string  // base64(scrypt(password, hashConfig))
  }
  user_metadata:     {}      // unused by this provider
  app_metadata:      object  // set by Auth Module when actor is assigned
}
```

The `password` field is **never** present in provider-to-framework return values — `upsertAuthIdentity` deep-copies the identity and deletes it before returning.

## 4. Password hashing

Algorithm: **scrypt** via `scrypt-kdf`.

```
hash = scrypt-kdf.kdf(password, { logN, r, p })
stored = hash.toString("base64")
```

Verification:
```
buf = Buffer.from(storedHash, "base64")
success = await scrypt-kdf.verify(buf, candidatePassword)
```

Default parameters `{ logN: 15, r: 8, p: 1 }` are chosen to be computationally expensive enough to resist offline dictionary attacks while remaining fast enough for production use (≈50–150 ms per hash on modern hardware).

## 5. Registration flow

```
register(userData, authIdentityService)
  1. Validate email (string) and password (string).
  2. Try authIdentityService.retrieve({ entity_id: email }).
     a. NOT_FOUND → create new ProviderIdentity (upsertAuthIdentity("create", ...))
     b. Found, app_metadata empty → update existing identity (upsertAuthIdentity("update", ...))
        → Supports the "invite flow": admin pre-creates an auth identity;
          first registration claims it.
     c. Found, app_metadata present → reject: "Identity with email already exists"
  3. Return { success, authIdentity } (password stripped).
```

## 6. Authentication flow

```
authenticate(userData, authIdentityService)
  1. Validate email and password are strings.
  2. authIdentityService.retrieve({ entity_id: email })
     → NOT_FOUND → return { success: false, error: "Invalid email or password" }
  3. Find ProviderIdentity for this.provider in authIdentity.provider_identities.
  4. Extract provider_metadata.password (base64 hash string).
  5. scrypt-kdf.verify(buf, password)
     → true  → return { success: true, authIdentity } (password stripped)
     → false → return { success: false, error: "Invalid email or password" }
```

The same error message is returned for "user not found" and "wrong password" to prevent email enumeration.

## 7. Update flow

```
update(data, authIdentityService)
  1. Require entity_id; reject without it.
  2. If no password provided, return { success: true } (no-op).
  3. Hash new password via hashPassword().
  4. authIdentityService.update(entity_id, { provider_metadata: { password: hash } })
  5. Return { success: true, authIdentity }.
```

## 8. JWT flow (delegated)

This provider returns `{ success: true, authIdentity }`. The Auth Module framework then:
1. Creates or retrieves the actor (customer/user).
2. Signs a JWT with the actor's `id` and `app_metadata`.
3. Returns the token to the HTTP layer.

The provider has no knowledge of JWTs.

## 9. Key design decisions

- **scrypt over bcrypt**: scrypt is memory-hard, making parallel GPU/ASIC attacks impractical. `logN: 15` is the recommended minimum for 2024.
- **No enumeration leakage**: Identical error messages for "not found" and "wrong password".
- **Invite-claim pattern**: An identity with empty `app_metadata` can be claimed on first registration, enabling invite flows.
- **Password stripped on return**: A deep copy + `delete` ensures the hash never leaks to callers.

## 10. Dependencies

| Package | Purpose |
|---|---|
| `scrypt-kdf` | Password hashing and verification |
| `@medusajs/framework` | `AbstractAuthModuleProvider`, `MedusaError` |
| `@medusajs/utils` | `isPresent` helper |
