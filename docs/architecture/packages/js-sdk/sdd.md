# Software Design Document — @medusajs/js-sdk

**Package:** `@medusajs/js-sdk`  
**Version:** 2.15.4  
**Status:** Stable  
**Owner:** Medusa Core Team

---

## 1. Purpose and Scope

`@medusajs/js-sdk` is the official HTTP client for the Medusa backend. It targets browser and Node.js environments and provides:

- A strongly typed, namespace-organised client for every Admin and Store API endpoint.
- Automatic JWT management (storage, refresh, clearing).
- Publishable API key injection for storefront requests.
- Server-Sent Events (SSE) streaming support.
- Zero non-native dependencies — only the platform `fetch` API is required.

---

## 2. Design Goals

| Goal | Decision |
|---|---|
| **No axios** | Uses native `fetch` for zero bundle overhead and edge runtime compatibility. |
| **Full type safety** | Every method is typed against `HttpTypes` from `@medusajs/framework/types`. |
| **Pluggable auth** | Supports JWT (localStorage / memory / custom) and session-cookie auth strategies interchangeably. |
| **Incremental adoption** | The `client.fetch` escape hatch allows calling any endpoint before it gets a named method. |
| **Storefront-ready** | Publishable key is injected automatically; no manual header management needed. |

---

## 3. Component Design

### 3.1 `Client` class

The `Client` class is the HTTP primitive. All named resource classes (`Admin`, `Store`) hold a reference to one `Client` instance.

```
Client
├── fetch<T>(input, init) → Promise<T>
│     - Serialises body to JSON
│     - Injects Authorization header (JWT or API key)
│     - Injects x-publishable-api-key header
│     - Throws FetchError on non-2xx status
│     - Parses JSON response when accept: application/json
├── fetchStream(input, init) → Promise<FetchStreamResponse>
│     - Returns async generator of ServerSentEventMessage
│     - Used for workflow execution progress events
├── setToken / getToken / clearToken  — token lifecycle
└── setLocale / getLocale             — per-request locale header
```

### 3.2 Token storage strategies

The JWT token storage is pluggable:

| Strategy | Behaviour |
|---|---|
| `"local"` (default) | Persists in `localStorage` under `jwtTokenStorageKey` |
| `"memory"` | Stored in a closure variable; lost on page reload |
| `"custom"` | Caller provides `{ get, set, remove }` callbacks |
| `"session"` auth type | No JWT; relies on `httpOnly` cookie set by the server |

### 3.3 `Admin` namespace

`Admin` is a container class that instantiates resource classes lazily:

```
Admin
├── product:          Product
├── order:            Order
├── customer:         Customer
├── customerGroup:    CustomerGroup
├── apiKey:           ApiKey
├── campaign:         Campaign
├── claim:            Claim
├── currency:         Currency
├── draftOrder:       DraftOrder
├── exchange:         Exchange
├── fulfillmentSet:   FulfillmentSet
├── inventoryItem:    InventoryItem
├── … (30+ resource classes)
└── Each resource class exposes: list, retrieve, create, update, delete
    plus domain-specific methods (e.g., product.import, order.cancel)
```

### 3.4 `Store` namespace

```
Store
├── product:     StoreProduct  → list, retrieve
├── cart:        StoreCart     → create, retrieve, createLineItem, updateLineItem,
│                                deleteLineItem, addShippingMethod, complete
├── customer:    StoreCustomer → create, retrieve, update
├── order:       StoreOrder    → retrieve, listOrders
├── collection:  StoreCollection
├── category:    StoreProductCategory
└── region:      StoreRegion
```

### 3.5 `Auth` namespace

```
Auth
├── login(actor, provider, body)   → JWT or redirect URL
├── logout()
├── session()                      → refresh session-cookie
├── callback(actor, provider, body) → handle OAuth callback
├── refreshToken()
├── mfa.*                          → TOTP setup, challenge, verify, recovery codes
└── register(actor, provider, body) → create credentials
```

### 3.6 `FetchError`

Non-2xx responses throw a `FetchError` instead of returning. This allows callers to use `try/catch` with `.status` discrimination:

```typescript
try {
  await sdk.admin.product.retrieve("not-found")
} catch (e) {
  if (e instanceof FetchError && e.status === 404) {
    // handle not found
  }
}
```

---

## 4. Request Lifecycle

```
sdk.admin.product.list(params)
  │
  ▼
Product.list()
  → builds URL: GET /admin/products?limit=20&...
  → calls client.fetch<AdminProductListResponse>(url, { method: "GET" })
       → injects Authorization: Bearer <token>
       → injects x-publishable-api-key (if configured)
       → native fetch()
       → status check → throws FetchError if non-2xx
       → JSON.parse(response)
  ← returns { products: ProductDTO[], count, limit, offset }
```

---

## 5. SSE / Streaming

Workflow execution events are streamed via Server-Sent Events:

```typescript
const { stream, abort } = await sdk.client.fetchStream(
  "/admin/workflows-executions/my-workflow/subscribe",
  { method: "GET" }
)

for await (const event of stream) {
  if (event.event === "workflow_done") {
    abort()
  }
}
```

---

## 6. Build Artifacts

The package ships both CJS (`dist/*.js`) and ESM (`dist/esm/*.js`) builds. The ESM build is used by bundlers (Vite, Next.js). The CJS build is used in Node.js environments.

---

## 7. Testing

- Unit tests mock `globalThis.fetch` using `jest.spyOn`.
- Auth flow tests verify token storage/retrieval across the three storage strategies.
- Integration tests in `dist/__tests__/` run against a live Medusa test server.

---

## 8. Open Questions / Future Work

- Add React Query / SWR integration helpers as a separate `@medusajs/react-sdk` package.
- Investigate automatic retry with exponential backoff for transient 5xx errors.
- Add OpenTelemetry `fetch` instrumentation for request tracing in Next.js apps.
