# SpecKit — @medusajs/auth-google

---

## 1. Unit specs — `authenticate` (initiate redirect)

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Happy path | Valid config | Returns `{ success: true, location: "https://accounts.google.com/o/oauth2/v2/auth?..." }` |
| U2 | `scope` in location | — | URL contains `scope=email+profile+openid` (or equivalent encoding) |
| U3 | `response_type=code` | — | URL contains `response_type=code` |
| U4 | `state` in location | — | URL contains `state=<32-byte hex>` |
| U5 | `redirect_uri` in location | — | URL contains `redirect_uri=<callbackUrl>` |
| U6 | State persisted in store | — | `authIdentityService.setState` called once |
| U7 | `body.callback_url` overrides | `body.callback_url: "https://alt.com/cb"` | State stored with overridden URL |
| U8 | Error in query | `query.error: "access_denied"` | Returns `{ success: false, error: "..., read more at: ..." }` |

---

## 2. Unit specs — `validateCallback`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U9 | Happy path — new user | Valid code, state, verified email | ProviderIdentity created; returns `{ success: true, authIdentity }` |
| U10 | Happy path — returning user | Same Google `sub` in DB | Existing ProviderIdentity retrieved; returns `{ success: true, authIdentity }` |
| U11 | Missing `code` | `query: { state: "abc" }` | Returns `{ success: false, error: "No code provided" }` |
| U12 | Missing / expired state | `query: { code: "x", state: "bad" }` | Returns `{ success: false, error: "No state provided, or session expired" }` |
| U13 | Token exchange failure | Google token endpoint returns 400 | Returns `{ success: false, error: "Could not exchange token, 400 ..." }` |
| U14 | `email_verified: false` in id_token | Unverified account | Returns `{ success: false, error: "Email not verified ..." }` |
| U15 | `entity_id` is `sub` | Valid flow | `authIdentity.provider_identities[0].entity_id === payload.sub` |
| U16 | `user_metadata` populated | Valid flow | Contains `name`, `email`, `picture`, `given_name`, `family_name` |

---

## 3. Unit specs — `verify_` (id_token decoding)

| # | Scenario | Expected outcome |
|---|---|---|
| U17 | Valid id_token, verified email | Returns `{ success: true, authIdentity }` |
| U18 | `email_verified: false` | Throws `MedusaError(INVALID_DATA)` |
| U19 | Null/undefined id_token | Returns `{ success: false, error: "No ID found" }` |

---

## 4. Unit specs — `register`

| # | Scenario | Expected outcome |
|---|---|---|
| U20 | Any input | Throws `MedusaError(NOT_ALLOWED, "Google does not support registration")` |

---

## 5. Unit specs — configuration validation

| # | Scenario | Expected outcome |
|---|---|---|
| U21 | `clientId` missing | Throws `Error("Google clientId is required")` |
| U22 | `clientSecret` missing | Throws `Error("Google clientSecret is required")` |
| U23 | `callbackUrl` missing | Throws `Error("Google callbackUrl is required")` |

---

## 6. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Full OAuth flow (mock Google) | Authenticate → redirect → callback → JWT issued |
| I2 | CSRF attack (wrong state) | Callback rejected |
| I3 | Unverified email | Authentication rejected at `verify_` |
| I4 | Returning user | Identity retrieved (not duplicated) |

---

## 7. Acceptance criteria

- CSRF state is cryptographically random (≥32 bytes entropy).
- `email_verified: false` always produces an error, never a successful auth.
- `entity_id` is always Google's `sub` (stable across sessions).
- `register` always throws `NOT_ALLOWED`.
- `access_token` is never persisted to the database.
