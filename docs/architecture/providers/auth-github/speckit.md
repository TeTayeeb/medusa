# SpecKit — @medusajs/auth-github

---

## 1. Unit specs — `authenticate` (initiate redirect)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Happy path | Valid config, no error params | Returns `{ success: true, location: "https://github.com/login/oauth/authorize?..." }` |
| U2 | Location includes client_id | — | `location` URL contains `client_id=<configured>` |
| U3 | Location includes state | — | `location` URL has `state=<hex32>` |
| U4 | Location includes redirect_uri | — | `location` URL has `redirect_uri=<callbackUrl>` |
| U5 | State persisted | — | `authIdentityService.setState` called with stateKey and `{ callback_url }` |
| U6 | `body.callback_url` overrides config | `body: { callback_url: "https://override.com/cb" }` | State stored with `callback_url: "https://override.com/cb"` |
| U7 | GitHub error in query | `query: { error: "access_denied", error_description: "..." }` | Returns `{ success: false, error: "..., read more at: ..." }` |

---

## 2. Unit specs — `validateCallback`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U8 | Happy path — new user | Valid code, state, GitHub returns user | ProviderIdentity created; returns `{ success: true, authIdentity }` |
| U9 | Happy path — returning user | Same GitHub user ID in DB | ProviderIdentity updated; returns `{ success: true, authIdentity }` |
| U10 | Missing code | `query: { state: "abc" }` | Returns `{ success: false, error: "No code provided" }` |
| U11 | Missing / expired state | `query: { code: "xyz", state: "unknown" }` | Returns `{ success: false, error: "No state provided, or session expired" }` |
| U12 | GitHub error param | `query: { error: "bad_verification_code" }` | Returns `{ success: false, error: "..., read more at: ..." }` |
| U13 | Token exchange HTTP failure | GitHub API returns 401 | Returns `{ success: false, error: "Could not exchange token, 401 ..." }` |
| U14 | `provider_metadata` contains access_token | Success flow | `authIdentity.provider_identities[0].provider_metadata.access_token` present |
| U15 | `user_metadata` contains name | Success flow | `authIdentity.user_metadata.name` set from GitHub profile |

---

## 3. Unit specs — `register`

| # | Scenario | Expected outcome |
|---|---|---|
| U16 | Called with any input | Throws `MedusaError(NOT_ALLOWED, "Github does not support registration")` |

---

## 4. Unit specs — configuration validation

| # | Scenario | Expected outcome |
|---|---|---|
| U17 | `clientId` missing | Throws `Error("Github clientId is required")` |
| U18 | `clientSecret` missing | Throws `Error("Github clientSecret is required")` |
| U19 | `callbackUrl` missing | Throws `Error("Github callbackUrl is required")` |
| U20 | All options provided | `validateOptions` returns without error |

---

## 5. Unit specs — `getRedirect`

| # | Scenario | Expected outcome |
|---|---|---|
| U21 | URL structure | Returns URL at `https://github.com/login/oauth/authorize` |
| U22 | All required params present | `client_id`, `redirect_uri`, `response_type=code`, `state` all set |

---

## 6. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Full OAuth flow (mock GitHub) | Authenticate → redirect → callback → JWT returned |
| I2 | CSRF attack (wrong state) | Callback with tampered state rejected |
| I3 | Repeat login same GitHub account | `entity_id` unchanged; `provider_metadata` refreshed |

---

## 7. Acceptance criteria

- CSRF state is cryptographically random (min 32 bytes of entropy).
- State is validated on every callback before any GitHub API call.
- `register` always throws `NOT_ALLOWED`.
- Token expiry timestamps are stored as ISO 8601 strings.
- `entity_id` is always the string form of the GitHub numeric user ID.
