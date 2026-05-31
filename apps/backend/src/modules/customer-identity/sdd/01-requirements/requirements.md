# customer-identity — Requirements

## Functional Requirements

| ID | Requirement |
|----|-------------|
| CID-01 | The module MUST support email/password customer login and logout. |
| CID-02 | The module MUST support customer registration (create account). |
| CID-03 | The module MUST expose a GET `/store/customers/me` endpoint for the authenticated customer's profile. |
| CID-04 | The module MUST manage JWT and cookie session lifecycle consistently across all API instances. |
| CID-05 | The module MUST NOT inspect `req.auth_context` manually — always use `AuthenticatedMedusaRequest`. |
| CID-06 | The module MUST be togglable via `FEATURE_CUSTOMER_IDENTITY_V2`. |
| CID-07 | The module MUST expose a health check returning `{ module: "customer-identity", status: "ok" }`. |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| CID-NF-01 | `JWT_SECRET` and `COOKIE_SECRET` MUST be identical across all `medusa-api` instances (multi-instance safe). |
| CID-NF-02 | Secrets MUST be rotated via environment variables — never hardcoded. |
| CID-NF-03 | Auth routes MUST honour `AUTH_CORS` for cross-origin preflight requests. |
| CID-NF-04 | The module MUST NOT import from Medusa private paths. |

## Out of Scope

- Social (OAuth) authentication (future extension via Medusa auth providers)
- Two-factor authentication (future)
- Admin user management (handled by Medusa's built-in admin auth)
