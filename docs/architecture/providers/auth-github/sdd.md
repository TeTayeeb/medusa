# Software Design Document — @medusajs/auth-github

## 1. Purpose

Enable GitHub OAuth 2.0 sign-in for Medusa actors (customers, admins). The provider manages the redirect/callback cycle, token exchange, and profile upsert. No registration form is involved — the GitHub account itself is the identity.

## 2. Architecture

```
Modules.AUTH
  └── ModuleProvider (auth-github)
        └── GithubAuthService (AbstractAuthModuleProvider)
              ├── authenticate(req, authIdentityService)   → redirect URL
              ├── validateCallback(req, authIdentityService) → authIdentity
              ├── register(_)                              → NOT_ALLOWED
              ├── upsert_(providerMetadata, svc)           → upsert ProviderIdentity
              └── getRedirect(clientId, callbackUrl, state) → { location }
```

## 3. State management (CSRF protection)

GitHub requires the `state` parameter to prevent CSRF attacks. The flow is:

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

The Auth Module's `setState`/`getState` uses a short-lived store (typically Redis or DB) to persist state between the redirect and callback. State is keyed by the random hex string, making collisions computationally infeasible.

## 4. Token exchange

```
POST https://github.com/login/oauth/access_token
  ?client_id=<clientId>
  &client_secret=<clientSecret>
  &code=<code>
  &redirect_uri=<callbackUrl>
Headers: Accept: application/json
```

Response:
```json
{
  "access_token": "gho_...",
  "refresh_token": "ghr_...",
  "expires_in": 28800,
  "refresh_token_expires_in": 15897600
}
```

Token expiry timestamps are calculated as `new Date(Date.now() + expiresIn * 1000).toISOString()`.

## 5. User profile fetch

After token exchange, the provider fetches:
```
GET https://api.github.com/user
Authorization: Bearer <access_token>
```

Relevant fields mapped to `user_metadata`:
| GitHub field | `user_metadata` key |
|---|---|
| `id` | used as `entity_id` (string) |
| `name` | `name` |
| `email` | `email` |
| `avatar_url` | `avatar` |
| `url` | `profile_url` |
| `company` | `company` |
| `two_factor_authentication` | `two_factor_authentication` |

## 6. ProviderIdentity upsert

```
upsert_(providerMetadata, authIdentityService):
  user = fetch GitHub /user API
  entity_id = user.id.toString()
  try:
    authIdentityService.update(entity_id, { provider_metadata, user_metadata })
  catch NOT_FOUND:
    authIdentityService.create({ entity_id, provider_metadata, user_metadata })
```

This is an upsert pattern: update if the GitHub user has authenticated before, create on first login.

## 7. Dynamic callback URL

The `callbackUrl` in options is the default. Callers may pass `body.callback_url` in the initial authenticate request to override it for that session. This allows multi-frontend deployments (storefront + admin) to use the same GitHub OAuth app.

## 8. Error handling

| Scenario | Response |
|---|---|
| GitHub `error` param in query | `{ success: false, error: "${description}, read more at: ${uri}" }` |
| Missing `code` in callback | `{ success: false, error: "No code provided" }` |
| State missing / expired | `{ success: false, error: "No state provided, or session expired" }` |
| Token exchange HTTP error | `MedusaError(INVALID_DATA, "Could not exchange token, {status}")` |
| `register` called | `MedusaError(NOT_ALLOWED, "Github does not support registration")` |

## 9. Security considerations

- **State parameter**: Cryptographic random; validated on callback. Prevents CSRF.
- **No token storage in JWTs**: `provider_metadata` lives in the DB, not in the JWT.
- **Token expiry stored**: Enables callers to implement refresh logic.
- **HTTPS required**: GitHub OAuth will not issue tokens over plain HTTP in production.

## 10. Dependencies

| Package | Purpose |
|---|---|
| `crypto` | Secure random state key generation |
| `@medusajs/framework` | `AbstractAuthModuleProvider`, `MedusaError` |
| `fetch` (Node 18+) | HTTP calls to GitHub API (no additional HTTP library) |
