# SpecKit — @medusajs/js-sdk

**Package:** `@medusajs/js-sdk`  
**Version:** 2.15.4  
**Document type:** Specification & Test Contracts  

---

## 1. Overview

This SpecKit defines the behavioural contracts for `@medusajs/js-sdk`. It covers HTTP request formation, authentication flows, error handling, token storage strategies, and SSE streaming.

---

## 2. Functional Specifications

### SPEC-JSSDK-01 — Correct URL and method for each resource operation

**Description:** Each resource method must produce the correct HTTP method and URL path.

| SDK call | Expected HTTP request |
|---|---|
| `sdk.admin.product.list({ limit: 5 })` | `GET /admin/products?limit=5` |
| `sdk.admin.product.retrieve("prod_123")` | `GET /admin/products/prod_123` |
| `sdk.admin.product.create({ title: "T" })` | `POST /admin/products` body: `{"title":"T"}` |
| `sdk.admin.product.update("prod_123", { title: "U" })` | `POST /admin/products/prod_123` |
| `sdk.admin.product.delete("prod_123")` | `DELETE /admin/products/prod_123` |
| `sdk.store.cart.create({ region_id: "r" })` | `POST /store/carts` |
| `sdk.store.cart.complete("cart_123")` | `POST /store/carts/cart_123/complete` |

---

### SPEC-JSSDK-02 — JWT token injection

**Description:** After a successful `sdk.auth.login`, every subsequent request must include `Authorization: Bearer <token>` in its headers.

**Contract:**
```typescript
// Mock fetch to capture headers
const calls: RequestInit[] = []
global.fetch = vi.fn(async (url, init) => {
  calls.push(init!)
  return new Response(JSON.stringify({ token: "jwt_abc" }))
})

await sdk.auth.login("user", "emailpass", { email: "a@b.com", password: "x" })

// Clear call history
calls.length = 0
global.fetch = vi.fn(async (url, init) => {
  calls.push(init!)
  return new Response(JSON.stringify({ products: [] }))
})

await sdk.admin.product.list({})
expect(calls[0].headers?.["Authorization"]).toBe("Bearer jwt_abc")
```

---

### SPEC-JSSDK-03 — Publishable API key injection

**Description:** When `publishableKey` is set in the SDK config, every Store request must include `x-publishable-api-key: <key>`.

**Contract:**
```typescript
const sdk = new Medusa({ baseUrl: "http://localhost:9000", publishableKey: "pk_test_123" })
// After any sdk.store.* call, the header must be present
```

Admin requests must NOT include the publishable API key header.

---

### SPEC-JSSDK-04 — `FetchError` on non-2xx responses

**Description:** Any non-2xx HTTP response must cause `client.fetch` to throw a `FetchError` with the correct `.status` and `.statusText`.

```typescript
global.fetch = vi.fn(async () =>
  new Response(JSON.stringify({ message: "Not found" }), { status: 404, statusText: "Not Found" })
)

await expect(sdk.admin.product.retrieve("missing")).rejects.toMatchObject({
  status: 404,
  statusText: "Not Found",
})
```

**Note:** 2xx responses (including 201, 204) must not throw.

---

### SPEC-JSSDK-05 — Token storage strategies

**Description:** Each storage strategy must persist and retrieve tokens correctly.

| Strategy | Set | Get | Clear |
|---|---|---|---|
| `"local"` | `localStorage.setItem(key, token)` | `localStorage.getItem(key)` | `localStorage.removeItem(key)` |
| `"memory"` | In-memory closure variable | Return closure variable | Set to `null` |
| `"custom"` | Calls `config.auth.storageFunction.set(key, token)` | Calls `config.auth.storageFunction.get(key)` | Calls `config.auth.storageFunction.remove(key)` |

**Edge case:** In a Node.js environment with `"local"` strategy, `localStorage` is not available. `client.getToken()` must return `undefined` gracefully (not throw).

---

### SPEC-JSSDK-06 — Session auth does not send Authorization header

**Description:** When `auth.type: "session"` is configured, `client.fetch` must not inject an `Authorization` header. Authentication is handled by browser cookies.

---

### SPEC-JSSDK-07 — `fetchStream` yields SSE events

**Description:** `client.fetchStream` must return an async generator that yields one object per SSE message, with `event` and `data` fields.

```typescript
// Given a server responding with:
// data: {"step":"step_1","status":"done"}\n\n
// event: workflow_done\ndata: {"result":"ok"}\n\n

const { stream, abort } = await sdk.client.fetchStream("/admin/ws/events")
const events: ServerSentEventMessage[] = []

for await (const msg of stream) {
  events.push(msg)
  if (msg.event === "workflow_done") abort()
}

expect(events[0].data).toBe('{"step":"step_1","status":"done"}')
expect(events[1].event).toBe("workflow_done")
```

---

### SPEC-JSSDK-08 — `setLocale` adds locale header

**Description:** After `sdk.setLocale("fr")`, every request must include `Accept-Language: fr` (or the configured locale header).

---

## 3. Non-Functional Specifications

### SPEC-JSSDK-NF-01 — Bundle size

**Contract:** The ESM build of `@medusajs/js-sdk` must not exceed **50 KB** (minified + gzip) when tree-shaken to include only `sdk.store.*` (storefront-only usage).

### SPEC-JSSDK-NF-02 — Edge runtime compatibility

**Contract:** The ESM build must pass a Cloudflare Workers compatibility check (`wrangler dev --compatibility-date=2024-01-01`) without polyfill errors.

---

## 4. Test Matrix

| Spec | Test type | Location |
|---|---|---|
| SPEC-JSSDK-01 | Unit (Vitest, fetch mock) | `dist/__tests__/admin/product.spec.ts` |
| SPEC-JSSDK-02 | Unit (Vitest, fetch mock) | `dist/__tests__/auth/jwt-injection.spec.ts` |
| SPEC-JSSDK-03 | Unit (Vitest, fetch mock) | `dist/__tests__/client/publishable-key.spec.ts` |
| SPEC-JSSDK-04 | Unit (Vitest, fetch mock) | `dist/__tests__/client/fetch-error.spec.ts` |
| SPEC-JSSDK-05 | Unit (Vitest) | `dist/__tests__/client/token-storage.spec.ts` |
| SPEC-JSSDK-06 | Unit (Vitest, fetch mock) | `dist/__tests__/auth/session-auth.spec.ts` |
| SPEC-JSSDK-07 | Unit (Vitest, ReadableStream mock) | `dist/__tests__/client/fetch-stream.spec.ts` |
| SPEC-JSSDK-08 | Unit (Vitest, fetch mock) | `dist/__tests__/client/locale.spec.ts` |

---

## 5. Glossary

| Term | Definition |
|---|---|
| **FetchError** | Error class thrown on non-2xx HTTP responses; carries `.status` and `.statusText` |
| **SSE** | Server-Sent Events — a one-directional streaming protocol over HTTP |
| **Publishable API key** | A public key (`pk_*`) that identifies a sales channel; safe to expose in frontend code |
| **JWT** | JSON Web Token — a signed token representing an authenticated session |
| **`fetchStream`** | SDK method for consuming SSE streams as an async generator |
