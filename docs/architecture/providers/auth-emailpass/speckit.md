# SpecKit — @medusajs/auth-emailpass

---

## 1. Unit specs — `register`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | New registration | `{ email: "a@b.com", password: "secret" }` | Returns `{ success: true, authIdentity }` with password stripped |
| U2 | Duplicate email, unclaimed identity (no app_metadata) | Existing identity, `app_metadata: null` | Updates password hash, returns `{ success: true }` |
| U3 | Duplicate email, claimed identity | Existing identity with `app_metadata` set | Returns `{ success: false, error: "Identity with email already exists" }` |
| U4 | Missing password | `{ email: "a@b.com" }` | Returns `{ success: false, error: "Password should be a string" }` |
| U5 | Missing email | `{ password: "secret" }` | Returns `{ success: false, error: "Email should be a string" }` |
| U6 | Password is non-string (number) | `{ email: "a@b.com", password: 1234 }` | Returns `{ success: false, error: "Password should be a string" }` |

---

## 2. Unit specs — `authenticate`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U7 | Correct credentials | Registered user, correct password | Returns `{ success: true, authIdentity }` |
| U8 | Wrong password | Registered user, wrong password | Returns `{ success: false, error: "Invalid email or password" }` |
| U9 | Unknown email | Email not in DB | Returns `{ success: false, error: "Invalid email or password" }` |
| U10 | Missing password | `{ email: "a@b.com" }` | Returns `{ success: false, error: "Password should be a string" }` |
| U11 | Missing email | `{ password: "secret" }` | Returns `{ success: false, error: "Email should be a string" }` |
| U12 | Password not in return value | Any successful auth | `authIdentity.provider_identities[0].provider_metadata` has no `password` field |

---

## 3. Unit specs — `update`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U13 | Update password | `{ entity_id: "a@b.com", password: "newpass" }` | Hash updated in DB; returns `{ success: true }` |
| U14 | Missing entity_id | `{ password: "newpass" }` | Returns `{ success: false, error: "Cannot update ... without entity_id" }` |
| U15 | No password field | `{ entity_id: "a@b.com" }` | No-op; returns `{ success: true }` |
| U16 | Password is non-string | `{ entity_id: "a@b.com", password: 123 }` | No-op; returns `{ success: true }` |

---

## 4. Unit specs — `hashPassword`

| # | Scenario | Expected outcome |
|---|---|---|
| U17 | Output is base64 string | `hashPassword("secret")` | Returns a base64-encoded string |
| U18 | Same input, different hashes | Two calls with identical password | Hashes differ (random salt embedded by scrypt-kdf) |
| U19 | Custom hashConfig applied | `options.hashConfig: { logN: 10, r: 8, p: 1 }` | Lower logN used (faster for tests) |

---

## 5. Unit specs — password stripping

| # | Scenario | Expected outcome |
|---|---|---|
| U20 | Register: password stripped from returned identity | Post-register response | `provider_metadata.password` is `undefined` |
| U21 | Authenticate: password stripped from returned identity | Post-auth response | `provider_metadata.password` is `undefined` |
| U22 | Original DB record not mutated | After strip | ProviderIdentity in DB still has `password` hash |

---

## 6. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Full register → authenticate flow | Register, then authenticate with same credentials | JWT returned after authentication |
| I2 | Register → authenticate with wrong password | — | Auth fails with generic error |
| I3 | Update password → authenticate with new password | — | Auth succeeds with new password |
| I4 | Update password → authenticate with old password | — | Auth fails |

---

## 7. Acceptance criteria

- `authenticate` takes < 500 ms with default `logN: 15` on CI hardware.
- Passwords are never present in any log output.
- Error messages for "unknown email" and "wrong password" are identical.
- `hashConfig` options are respected when supplied.
