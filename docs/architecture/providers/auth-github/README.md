# @medusajs/auth-github

GitHub OAuth 2.0 authentication provider for Medusa v2. Implements `IAuthProvider` via `AbstractAuthModuleProvider` and registers under `Modules.AUTH` with identifier `github`.

The provider implements the **Authorization Code** OAuth flow: redirect to GitHub → exchange code for token → fetch GitHub user profile → upsert `ProviderIdentity`.

## Installation

```bash
npm install @medusajs/auth-github
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
            resolve: "@medusajs/auth-github",
            id: "github",
            options: {
              clientId: process.env.GITHUB_CLIENT_ID,
              clientSecret: process.env.GITHUB_CLIENT_SECRET,
              callbackUrl: process.env.GITHUB_CALLBACK_URL,
              // e.g. "https://your-storefront.com/auth/github/callback"
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
| `clientId` | `string` | ✅ | GitHub OAuth App Client ID |
| `clientSecret` | `string` | ✅ | GitHub OAuth App Client Secret |
| `callbackUrl` | `string` | ✅ | Redirect URI registered in GitHub OAuth App settings |

Configuration errors throw at boot time with a descriptive message.

## OAuth flow

### Step 1 — Initiate (`authenticate`)

```
GET /auth/customer/github
```

1. Generates a cryptographically random 32-byte `stateKey` (hex).
2. Persists `{ callback_url }` in the Auth Module's state store (keyed by `stateKey`).
3. Returns `{ success: true, location: <githubAuthUrl> }`.
4. The caller redirects the browser to `location`.

The constructed GitHub URL is:
```
https://github.com/login/oauth/authorize
  ?client_id=<clientId>
  &redirect_uri=<callbackUrl>
  &response_type=code
  &state=<stateKey>
```

### Step 2 — Callback (`validateCallback`)

```
GET /auth/customer/github/callback?code=<code>&state=<stateKey>
```

1. Validates `code` and `state` are present.
2. Retrieves and validates state from Auth Module store (prevents CSRF).
3. Exchanges `code` for `access_token` via `POST https://github.com/login/oauth/access_token`.
4. Fetches GitHub user profile via `GET https://api.github.com/user` (Bearer token).
5. Upserts `ProviderIdentity` keyed by GitHub user numeric `id` (as string).
6. Stores in `provider_metadata`: `access_token`, `refresh_token`, token expiry timestamps.
7. Stores in `user_metadata`: `name`, `email`, `avatar`, `company`, `profile_url`, `two_factor_authentication`.

## Provider identity structure

```ts
{
  entity_id: "12345678",               // GitHub user numeric ID
  provider: "github",
  provider_metadata: {
    access_token: "gho_...",
    refresh_token: "ghr_...",
    access_token_expires_at: "2025-...",
    refresh_token_expires_at: "2026-...",
  },
  user_metadata: {
    name: "Jane Doe",
    email: "jane@example.com",
    avatar: "https://avatars.githubusercontent.com/...",
    company: "Acme",
    profile_url: "https://api.github.com/users/janedoe",
    two_factor_authentication: true,
  }
}
```

## Environment variables

```dotenv
GITHUB_CLIENT_ID=Ov23li...
GITHUB_CLIENT_SECRET=abc123...
GITHUB_CALLBACK_URL=https://yourstore.com/auth/github/callback
```

## Notes

- `register` is **not supported** — throws `NOT_ALLOWED`. GitHub users authenticate directly via OAuth.
- If a callback URL is passed in the request body (`body.callback_url`), it overrides the configured `callbackUrl` for that session only.
