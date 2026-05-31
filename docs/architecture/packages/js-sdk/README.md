# @medusajs/js-sdk

> Version: 2.15.4 · Package path: `packages/core/js-sdk`

The **js-sdk** package is the official JavaScript/TypeScript HTTP client for the Medusa backend. It wraps the native `fetch` API with full type safety, automatic JWT management, publishable API key support, and a fluent namespace-based interface for every Admin and Store endpoint. It is used by the Medusa admin dashboard, and is the recommended way to integrate any storefront or third-party application with Medusa.

---

## Installation

```bash
pnpm add @medusajs/js-sdk
```

---

## Quick Start

```typescript
import Medusa from "@medusajs/js-sdk"

const sdk = new Medusa({
  baseUrl: "https://api.my-store.com",
  publishableKey: "pk_live_...",
  auth: {
    type: "session",   // or "jwt"
  },
})
```

---

## Authentication

### Email / Password login (Admin)

```typescript
const { token } = await sdk.auth.login("user", "emailpass", {
  email: "admin@example.com",
  password: "supersecret",
})
// Token is cached automatically for subsequent requests
```

### Email / Password login (Customer)

```typescript
await sdk.auth.login("customer", "emailpass", {
  email: "customer@example.com",
  password: "mypassword",
})
```

### OAuth / Social login

```typescript
// Step 1: get redirect URL
const { location } = await sdk.auth.login("customer", "google", {})
window.location.href = location

// Step 2: handle callback
const { token } = await sdk.auth.callback("customer", "google", {
  code: searchParams.get("code"),
  state: searchParams.get("state"),
})
```

### Session-based auth

```typescript
const sdk = new Medusa({ baseUrl: "...", auth: { type: "session" } })
await sdk.auth.session()   // refreshes the session
await sdk.auth.logout()    // invalidates the session
```

---

## Admin API

### Products

```typescript
// List products with filters
const { products, count } = await sdk.admin.product.list({
  status: ["published"],
  limit: 20,
  offset: 0,
  fields: "id,title,thumbnail,variants.id,variants.sku",
})

// Retrieve a single product
const { product } = await sdk.admin.product.retrieve("prod_01HX...", {
  fields: "+variants,+options",
})

// Create a product
const { product: created } = await sdk.admin.product.create({
  title: "New Shirt",
  status: "draft",
  variants: [{ title: "S / Blue", prices: [{ amount: 1999, currency_code: "usd" }] }],
})

// Update
await sdk.admin.product.update("prod_01HX...", { status: "published" })

// Delete
await sdk.admin.product.delete("prod_01HX...")
```

### Orders

```typescript
const { orders } = await sdk.admin.order.list({ status: ["pending"] })
const { order }  = await sdk.admin.order.retrieve("order_01HX...")
await sdk.admin.order.cancel("order_01HX...")
```

### Customers

```typescript
const { customers } = await sdk.admin.customer.list({ limit: 50 })
const { customer }  = await sdk.admin.customer.create({
  email: "new@example.com",
  first_name: "Jane",
  last_name: "Doe",
})
```

---

## Store API

### Products

```typescript
const { products } = await sdk.store.product.list({
  region_id: "reg_01HX...",
  limit: 12,
})
```

### Cart

```typescript
// Create cart
const { cart } = await sdk.store.cart.create({
  region_id: "reg_01HX...",
  items: [{ variant_id: "variant_01HX...", quantity: 1 }],
})

// Add item
await sdk.store.cart.createLineItem(cart.id, {
  variant_id: "variant_01HX...",
  quantity: 2,
})

// Complete cart (checkout)
const { type, order } = await sdk.store.cart.complete(cart.id)
```

---

## Low-level `client.fetch`

For endpoints not yet covered by named methods:

```typescript
const data = await sdk.client.fetch<{ custom: string }>("/custom/route", {
  method: "POST",
  body: { key: "value" },
  headers: { "x-my-header": "foo" },
})
```

---

## Configuration Reference

```typescript
new Medusa({
  baseUrl: "https://api.example.com",   // required
  publishableKey: "pk_...",              // for Store requests
  apiKey: "sk_...",                      // for Admin requests (API key auth)
  auth: {
    type: "jwt" | "session",
    jwtTokenStorageKey: "my_token",      // localStorage key override
    jwtTokenStorageMethod: "local" | "memory" | "custom",
  },
  logger: {
    error: console.error,
    warn: console.warn,
    debug: console.debug,
  },
})
```

---

## See Also

- [`@medusajs/framework/types`](../types/README.md) — `HttpTypes` used for all request/response typing
- [Storefront guide](https://docs.medusajs.com/learn/storefront-development)
- [JS SDK reference](https://docs.medusajs.com/sdk)
