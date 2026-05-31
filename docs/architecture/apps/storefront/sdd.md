# @dtc/storefront — System Design Document

> Version: 1.0 | Next.js: 15.5.18 | React: 19 | Medusa SDK: 2.15.3 | Status: Current

---

## 1. Purpose and Scope

This document describes the internal design of `@dtc/storefront` — the Next.js reference storefront for the DTC platform. It covers the App Router architecture, page rendering strategies, Medusa SDK integration, cart state management, checkout flow, and authentication model.

---

## 2. Next.js App Router Architecture

The storefront uses **Next.js 15 App Router** exclusively. There is no `pages/` directory. All routes are file-system-based React Server Components (RSC) by default, with `"use client"` directives applied only where interactivity is required.

### Route Group Organisation

```
app/
└── [countryCode]/          # Dynamic segment — resolved by middleware.ts
    ├── (main)/             # Route group: public storefront
    └── (checkout)/         # Route group: full-screen checkout (no nav/footer)
```

The `[countryCode]` segment is transparent to the URL structure from the user's perspective — it is injected by `middleware.ts` before the route is resolved, enabling region-specific pricing, currency, and locale without URL changes visible in the browser.

### Middleware Region Resolution

```typescript
// middleware.ts (simplified)
export async function middleware(request: NextRequest) {
  const country = getCountryFromCookie(request) ?? detectFromHeaders(request)
  const pathname = request.nextUrl.pathname
  if (!pathname.startsWith(`/${country}`)) {
    return NextResponse.redirect(new URL(`/${country}${pathname}`, request.url))
  }
}
```

---

## 3. Page Rendering Strategies

| Page | Strategy | Rationale |
|---|---|---|
| Homepage | **SSR** (dynamic) | Hero + featured collections need fresh data |
| Store listing | **SSR + Streaming** | Suspense boundaries per section; large product sets |
| Product detail `/products/[handle]` | **SSG + ISR** via `generateStaticParams` | Maximum SEO performance; revalidated on-demand |
| Collection / Category | **SSR** | Filtered product sets; vary by query params |
| Cart `/cart` | **CSR** | Cart is session-local; no SSR benefit |
| Checkout `/checkout` | **CSR** | Interactive multi-step form; Stripe Elements |
| Account `/account` | **CSR** | Auth-gated; parallel route swap on login |
| Order confirmation | **SSR** | One-time render; shareable confirmation URL |

---

## 4. Medusa SDK Integration

### Initialisation (`src/lib/config.ts`)

```typescript
import Medusa, { FetchArgs, FetchInput } from "@medusajs/js-sdk"
import { getLocaleHeader } from "@lib/util/get-locale-header"

let MEDUSA_BACKEND_URL = "http://localhost:9000"
if (process.env.NEXT_PUBLIC_MEDUSA_BACKEND_URL) {
  MEDUSA_BACKEND_URL = process.env.NEXT_PUBLIC_MEDUSA_BACKEND_URL
}

export const sdk = new Medusa({
  baseUrl: MEDUSA_BACKEND_URL,
  debug: process.env.NODE_ENV === "development",
  publishableKey: process.env.NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY,
})

// Patch sdk.client.fetch to inject x-medusa-locale header
const originalFetch = sdk.client.fetch.bind(sdk.client)
sdk.client.fetch = async <T>(input: FetchInput, init?: FetchArgs): Promise<T> => {
  const localeHeader = await getLocaleHeader()
  init = { ...init, headers: { ...localeHeader, ...(init?.headers ?? {}) } }
  return originalFetch(input, init)
}
```

The `sdk` singleton is imported in **all data-fetching functions** within `src/lib/data/`. On the server side (RSC / Route Handlers), `server-only` guards prevent the SDK from being bundled into client code unintentionally.

### Data Layer (`src/lib/data/`)

Each file exports async functions that call the Medusa SDK and return typed data:

| File | Key Exports |
|---|---|
| `products.ts` | `getProductByHandle()`, `listProducts()`, `getProductsList()` |
| `cart.ts` | `retrieveCart()`, `addToCart()`, `updateLineItem()`, `deleteLineItem()`, `setCartAddress()` |
| `customer.ts` | `getCustomer()`, `login()`, `logout()`, `register()`, `updateCustomer()` |
| `payment.ts` | `initiatePaymentSession()`, `placeOrder()` |
| `fulfillment.ts` | `listCartShippingMethods()`, `setShippingMethod()` |
| `orders.ts` | `retrieveOrder()`, `listOrders()` |
| `regions.ts` | `listRegions()`, `getRegion()` |
| `cookies.ts` | Cart ID and auth token cookie helpers (server-side only) |

---

## 5. Cart State Management

Cart state is managed **server-side first** with client-side optimistic updates:

1. The cart ID is stored in a **signed HTTP-only cookie** (`_medusa_cart_id`).
2. On each page load, the cart is fetched server-side in the layout and passed as a prop/context value.
3. Client-side mutations (add, update, remove) call **Next.js Server Actions** which in turn call the Medusa SDK, then revalidate the layout cache via `revalidatePath()` / `revalidateTag()`.
4. The `modal-context.tsx` (`src/lib/context/`) manages cart drawer open/close state.

### Cart Flow

```
User clicks "Add to Cart"
    → Client Component fires Server Action
    → Server Action calls sdk.store.cart.lineItems.create()
    → Server Action calls revalidatePath("/[countryCode]/cart")
    → Next.js re-renders affected Server Components
    → Cart item count badge updates
```

---

## 6. Checkout Flow

The checkout is a **single-page multi-step flow** implemented entirely client-side under `(checkout)/checkout/page.tsx`:

```
Step 1: Email / Contact
    ↓
Step 2: Shipping Address   ← sdk.store.cart.update({ shipping_address })
    ↓
Step 3: Delivery Method    ← sdk.store.cart.shippingMethods.create()
    ↓
Step 4: Payment            ← sdk.store.payment.initiate() → Stripe PaymentElement
    ↓
Step 5: Review & Confirm
    ↓
Step 6: Place Order        ← sdk.store.cart.complete() → sdk.store.order.confirm()
    ↓
Redirect → /order/[id]/confirmed
```

### Payment Integration

Payment is handled via **Stripe Elements** (`@stripe/react-stripe-js`). The Medusa backend returns a `client_secret` when a payment session is initiated; the storefront mounts `<PaymentElement>` with that secret, then calls `stripe.confirmPayment()` before placing the order. Alternative payment providers follow the same interface through Medusa's payment module abstraction.

---

## 7. Authentication Flow

```
/account (parallel route)
├── @login  (default — unauthenticated)
└── @dashboard  (authenticated)
    ├── /profile
    ├── /addresses
    └── /orders / /orders/details/[id]
```

Authentication state is determined server-side:

```typescript
// Simplified — src/modules/account/templates/account-layout.tsx
const customer = await getCustomer().catch(() => null)
return customer ? <Dashboard /> : <Login />
```

Login submits credentials via a Server Action → `sdk.auth.emailpass.authenticate()` → stores the returned JWT in a **secure session cookie** → the layout re-renders showing the dashboard slot.

---

## 8. Module Organisation

Feature modules in `src/modules/` each contain:

```
<module>/
├── components/   # Presentational React components
└── templates/    # Page-level composed templates (assembled from components)
```

Templates are imported directly into `page.tsx` files, keeping pages thin:

```typescript
// app/[countryCode]/(main)/products/[handle]/page.tsx
import ProductTemplate from "@modules/products/templates"
export default async function ProductPage({ params }) {
  const product = await getProductByHandle(params.handle, params.countryCode)
  return <ProductTemplate product={product} region={region} />
}
```

---

## 9. Internationalisation & Region Awareness

- Every SDK call that returns pricing data passes the region ID (derived from `countryCode`).
- The `x-medusa-locale` header is injected on every SDK request for locale-aware content.
- Currency formatting is handled by `Intl.NumberFormat` scoped to the active region's currency code.
- Country selection triggers a cookie update + page reload to re-enter the `[countryCode]` segment.
