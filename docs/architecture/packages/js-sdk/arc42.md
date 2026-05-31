# arc42 Architecture Documentation — @medusajs/js-sdk

**Package:** `@medusajs/js-sdk`  
**Version:** 2.15.4  
**Template:** arc42 v8  

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@medusajs/js-sdk` must:
- Provide a typed HTTP client usable in browsers, Node.js, and edge runtimes.
- Handle authentication transparently so application code focuses on data, not headers.
- Surface all Admin and Store endpoints as discoverable, auto-completing TypeScript methods.
- Support real-time features (SSE) without requiring WebSockets.

### 1.2 Quality Goals

| Priority | Quality | Scenario |
|---|---|---|
| 1 | **Developer experience** | An IDE shows autocomplete for every endpoint parameter with full types. |
| 2 | **Portability** | The SDK works identically in Next.js App Router (edge), Vite, and Node.js scripts. |
| 3 | **Security** | Tokens are never exposed in URLs; they are injected via headers only. |
| 4 | **Maintainability** | Adding a new API endpoint requires adding one file under `admin/` or `store/`. |

---

## 2. Architecture Constraints

- No axios or any non-native HTTP library.
- No Node.js-specific APIs (`http`, `https`, `fs`) in the main build — edge runtime must work.
- All request/response types must come from `@medusajs/framework/types` (`HttpTypes`), not be defined locally.

---

## 3. System Scope and Context

```
┌──────────────┐          ┌──────────────────────┐         ┌─────────────────┐
│  Storefront  │──────── ▶│    @medusajs/js-sdk  │────────▶│  Medusa Backend │
│  Admin UI    │          │  sdk.admin.*          │  HTTPS  │  /admin/*       │
│  Node script │          │  sdk.store.*          │         │  /store/*       │
└──────────────┘          │  sdk.auth.*           │         │  /auth/*        │
                          └──────────────────────┘         └─────────────────┘
```

---

## 4. Solution Strategy

### 4.1 Namespace-based resource organisation

All resources are grouped under `sdk.admin`, `sdk.store`, and `sdk.auth`. This mirrors the URL structure of the API (`/admin/products` → `sdk.admin.product`) and aids discoverability.

### 4.2 Thin resource classes

Each resource class (e.g., `Product`) holds a single `client` reference and delegates all HTTP calls to `client.fetch`. The resource class is responsible only for:
- Building the correct URL.
- Typing the request body and response.
- Calling `client.fetch` with the right method and parameters.

No caching, retry, or business logic lives in resource classes.

### 4.3 Auth abstraction

Auth is separated into its own `Auth` class rather than being mixed into `Admin` or `Store`. This reflects reality: auth endpoints have a different flow (they produce credentials, not data) and may be used independently of the resource APIs.

### 4.4 Pluggable token storage

The token storage strategy is resolved at construction time and stored on the `Client` instance. This avoids conditionals in hot paths:

```typescript
// Construction-time resolution
const storage = resolveStorage(config.auth?.jwtTokenStorageMethod)
// Hot path: always calls the same interface
await storage.set(key, token)
```

---

## 5. Building Block View

```
js-sdk/
├── index.ts            Medusa class (entry point), re-exports
├── client.ts           Client class, FetchError, PUBLISHABLE_KEY_HEADER
├── types.ts            Config, ClientHeaders, FetchArgs, FetchInput, Logger, …
├── auth/
│   └── index.ts        Auth class (login, logout, session, mfa.*, callback, register)
├── admin/
│   ├── index.ts        Admin class — instantiates all admin resource classes
│   ├── product.ts      Product class (list, retrieve, create, update, delete, import, export)
│   ├── order.ts        Order class
│   ├── customer.ts     Customer class
│   ├── … (30+ files)
│   └── index.ts        barrel export of Admin
└── store/
    ├── index.ts        Store class — instantiates all store resource classes
    ├── product.ts
    ├── cart.ts
    ├── customer.ts
    └── index.ts
```

---

## 6. Runtime View

### Admin product list call

```
developer calls: sdk.admin.product.list({ limit: 10 })
  │
  ▼
Product.list(params)
  builds URL: "/admin/products?limit=10"
  calls: this.client.fetch(url, { method: "GET" })
    │
    ▼
  Client.fetch()
    injects: Authorization: Bearer <stored token>
    injects: x-publishable-api-key (if set)
    calls:   globalThis.fetch(url, requestInit)
    awaits:  response
    checks:  response.ok → throws FetchError if false
    parses:  response.json()
    returns: { products: [...], count: N, limit: 10, offset: 0 }
```

### JWT refresh flow

```
client.fetch() receives 401
  │
  ├─ calls: client.refreshToken()
  │    GET /auth/token/refresh
  │    stores new token
  │
  └─ retries original request with new token
```

---

## 7. Deployment View

The SDK ships as:
- **CJS** (`dist/*.js`) — consumed by Node.js / bundler-unaware environments.
- **ESM** (`dist/esm/*.js`) — consumed by Vite, Next.js App Router, and modern bundlers.

The package.json `exports` map selects the correct build based on the consumer's module system.

---

## 8. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | Native `fetch` only | Works in edge runtimes (Cloudflare Workers, Vercel Edge); no polyfill needed in modern environments. |
| AD-02 | Namespace-per-resource classes | Organises 30+ resources without a single monolithic file; IDE autocomplete works per namespace. |
| AD-03 | Types from `HttpTypes` | Prevents type drift between server and client; one change propagates everywhere. |
| AD-04 | `FetchError` instead of rejection codes | Gives callers a structured `.status` to branch on without needing response parsing. |

---

## 9. Risks and Technical Debt

- **Missing named methods** — new endpoints must be manually added as resource class methods. The gap between API release and SDK update is a UX pain point; `client.fetch` is the stop-gap.
- **No retry logic** — transient 5xx errors require manual retry in caller code; automatic retry with backoff is planned.
- **Token storage on server** — `localStorage` is only available in browsers; server-side usage must use `"memory"` or `"custom"` storage. This is a common footgun.
