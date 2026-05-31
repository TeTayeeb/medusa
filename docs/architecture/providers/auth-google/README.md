# @medusajs/auth-google

Google OAuth 2.0 authentication provider for Medusa v2. Implements `IAuthProvider` via `AbstractAuthModuleProvider` and registers under `Modules.AUTH` with identifier `google`.

Uses the **Authorization Code** flow with Google's OpenID Connect layer: the callback exchanges the authorization code for an `id_token` (JWT), which is decoded to extract the user's verified Google profile — no separate profile API call required.

## Installation

```bash
npm install @medusajs/auth-google
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
            resolve: "@medusajs/auth-google",
            id: "google",
            options: {
              clientId: process.env.GOOGLE_CLIENT_ID,
              clientSecret: process.env.GOOGLE_CLIENT_SECRET,
              callbackUrl: process.env.GOOGLE_CALLBACK_URL,
              // e.g. "https://yourstore.com/auth/google/callback"
            },
          },
        ],
      },
    },
  ],
})
```

### Options reference

| Option | Type | Required | Description |
|---|---|---|---|
| `clientId` | `string` | ✅ | Google OAuth 2.0 Client ID |
| `clientSecret` | `string` | ✅ | Google OAuth 2.0 Client Secret |
| `callbackUrl` | `string` | ✅ | Authorised redirect URI (must be registered in Google Cloud Console) |

Missing options throw at boot time.

## OAuth flow

### Step 1 — Initiate (`authenticate`)

```
GET /auth/customer/google
```

1. Generates a 32-byte cryptographic `stateKey`.
2. Saves `{ callback_url }` in the Auth Module state store.
3. Returns `{ success: true, location: <googleAuthUrl> }`.

Google redirect URL:
```
https://accounts.google.com/o/oauth2/v2/auth
  ?client_id=<clientId>
  &redirect_uri=<callbackUrl>
  &response_type=code
  &scope=email profile openid
  &state=<stateKey>
```

### Step 2 — Callback (`validateCallback`)

```
GET /auth/customer/google/callback?code=<code>&state=<stateKey>
```

1. Validates `code` and `state`.
2. Validates state from Auth Module store (CSRF guard).
3. Exchanges `code` for tokens via `POST https://oauth2.googleapis.com/token`.
4. Decodes the returned `id_token` (Google-signed JWT) using `jsonwebtoken.decode`.
5. Validates `email_verified === true`; throws `INVALID_DATA` if not.
6. Upserts `ProviderIdentity` keyed by Google subject ID (`sub`).

## Provider identity structure

```ts
{
  entity_id: "1065842...",    // Google "sub" (subject), stable unique user ID
  provider: "google",
  provider_metadata: {},      // Not used by this provider
  user_metadata: {
    name: "Jane Doe",
    email: "jane@gmail.com",
    picture: "https://lh3.googleusercontent.com/...",
    given_name: "Jane",
    family_name: "Doe",
  }
}
```

> **Note**: Unlike the GitHub provider, Google stores nothing in `provider_metadata` — the `access_token` from Google is not persisted. Only the profile from the `id_token` is stored.

## Key differences vs auth-github

| Feature | auth-github | auth-google |
|---|---|---|
| Profile retrieval | Separate API call | Decoded from `id_token` |
| Token storage | `access_token` + `refresh_token` in `provider_metadata` | Nothing stored |
| Email verification enforced | No | Yes (`email_verified` must be `true`) |
| Scope | Read-only profile | `email profile openid` |

## Environment variables

```dotenv
GOOGLE_CLIENT_ID=123456789-abc.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-...
GOOGLE_CALLBACK_URL=https://yourstore.com/auth/google/callback
```

## Notes

- `register` is **not supported** — throws `NOT_ALLOWED`.
- Users with unverified Google email addresses are rejected.
- `body.callback_url` in the initiate request overrides the configured `callbackUrl` for that session.
