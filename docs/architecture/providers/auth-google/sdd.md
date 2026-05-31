# Software Design Document — @medusajs/auth-google

## 1. Purpose

Enable Google Sign-In (OAuth 2.0 + OpenID Connect) for Medusa actors. The provider manages the redirect/callback OAuth cycle. It uses the `id_token` returned in the token exchange (OpenID Connect) to obtain verified user profile data without an additional profile API call.

## 2. Architecture

```
Modules.AUTH
  └── ModuleProvider (auth-google)
        └── GoogleAuthService (AbstractAuthModuleProvider)
              ├── authenticate(req, authIdentityService)    → redirect URL
              ├── validateCallback(req, authIdentityService) → authIdentity
              ├── register(_)                               → NOT_ALLOWED
              ├── verify_(idToken, authIdentityService)     → upsert identity
              └── getRedirect(clientId, callbackUrl, state) → { location }
```

## 3. CSRF protection via state parameter

Identical pattern to auth-github:

```
authenticate():
  stateKey = crypto.randomBytes(32).toString("hex")
  authIdentityService.setState(stateKey, { callback_url })
  → redirect with state=stateKey

validateCallback():
  state = authIdentityService.getState(query.state)
  if (!state) → reject: "No state provided, or session expired"
  → proceed with code exchange
```

## 4. Authorization Code exchange

```
POST https://oauth2.googleapis.com/token
  ?client_id=<clientId>
  &client_secret=<clientSecret>
  &code=<code>
  &redirect_uri=<callbackUrl>
  &grant_type=authorization_code
```

Response:
```json
{
  "access_token": "ya29.xxx",
  "id_token": "<JWT>",
  "token_type": "Bearer",
  "expires_in": 3599,
  ...
}
```

Only `id_token` is used downstream; `access_token` is not persisted.

## 5. id_token decoding (OpenID Connect)

```
verify_(idToken, authIdentityService):
  jwtData = jsonwebtoken.decode(idToken, { complete: true })
  payload = jwtData.payload
  if (!payload.email_verified) → throw MedusaError(INVALID_DATA, "Email not verified")
  entity_id = payload.sub   // stable Google unique user ID
  userMetadata = {
    name, email, picture, given_name, family_name
  }
```

> **No signature verification**: `jsonwebtoken.decode` is used (not `verify`). The token's authenticity is implicitly trusted because it was obtained directly from `https://oauth2.googleapis.com/token` over HTTPS using the application's own `client_secret`. A MITM would require compromising the TLS channel.

## 6. Email verification enforcement

```
if (!payload.email_verified) {
  throw new MedusaError(
    MedusaError.Types.INVALID_DATA,
    "Email not verified, cannot proceed with authentication"
  )
}
```

This is a hard guard. Google accounts with unverified email (rare — typically G Suite accounts) cannot authenticate.

## 7. ProviderIdentity upsert

```
verify_(idToken, authIdentityService):
  try:
    authIdentity = authIdentityService.retrieve({ entity_id })
    // returning user — identity already exists, no update (user_metadata not refreshed)
  catch NOT_FOUND:
    authIdentity = authIdentityService.create({ entity_id, user_metadata })
    // first login — create new identity
  return { success: true, authIdentity }
```

> **Note**: Unlike auth-github which calls `update` for returning users (to refresh tokens), auth-google only creates on first login and retrieves on subsequent logins. `user_metadata` is not refreshed.

## 8. Scopes requested

`scope=email profile openid` — the minimum set required to obtain `email`, `name`, `picture`, `sub`, `given_name`, `family_name` in the `id_token`.

## 9. No token storage

`access_token` and `refresh_token` are not stored in `provider_metadata`. This is a deliberate simplification:
- The provider is stateless after the callback.
- Profile data is read once and cached in `user_metadata`.
- If a use case requires calling Google APIs on behalf of the user, a custom extension would be needed.

## 10. Error handling

| Scenario | Response |
|---|---|
| `error` in query params | `{ success: false, error: "..., read more at: ..." }` |
| Missing `code` | `{ success: false, error: "No code provided" }` |
| State missing/expired | `{ success: false, error: "No state provided, or session expired" }` |
| Token exchange HTTP failure | `MedusaError(INVALID_DATA, "Could not exchange token, {status}")` |
| `email_verified: false` | `MedusaError(INVALID_DATA, "Email not verified")` |
| `register` called | `MedusaError(NOT_ALLOWED, "Google does not support registration")` |

## 11. Dependencies

| Package | Purpose |
|---|---|
| `jsonwebtoken` | `decode` Google's `id_token` to extract payload |
| `crypto` | Secure random state key generation |
| `@medusajs/framework` | `AbstractAuthModuleProvider`, `MedusaError` |
| `fetch` (Node 18+) | Token exchange with Google |
