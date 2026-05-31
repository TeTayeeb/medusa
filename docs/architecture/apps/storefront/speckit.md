# @dtc/storefront — SpecKit

> Specification inventory for the Next.js storefront application
> Version: 1.0 | Next.js: 15.5.18 | Medusa SDK: 2.15.3

---

## 1. Overview

This document provides a structured specification inventory for `@dtc/storefront`. It defines functional requirements, page specifications, component contracts, data-fetching interfaces, and acceptance criteria for the customer-facing commerce frontend.

---

## 2. Functional Specifications

### 2.1 Region & Locale

| ID | Requirement |
|---|---|
| RL-01 | System SHALL detect and redirect users to their regional URL (`/[countryCode]/…`) via edge middleware |
| RL-02 | System SHALL support country codes: `gb`, `de`, `dk`, `se`, `fr`, `es`, `it` |
| RL-03 | System SHALL persist the selected region in a cookie (`_medusa_country`) |
| RL-04 | System SHALL inject `x-medusa-locale` header on every SDK request |
| RL-05 | System SHALL display prices in the region's currency and format |

---

### 2.2 Product Catalogue

| ID | Requirement |
|---|---|
| PC-01 | Product detail pages SHALL be statically generated for all published products |
| PC-02 | Product images SHALL use Next.js `<Image>` with `priority` on the hero image |
| PC-03 | Variant selection SHALL update URL query params without navigation |
| PC-04 | Out-of-stock variants SHALL be visually disabled; "Add to cart" SHALL be disabled |
| PC-05 | Product pages SHALL include OpenGraph metadata for social sharing |
| PC-06 | Collection and category pages SHALL support pagination or infinite scroll |

---

### 2.3 Cart

| ID | Requirement |
|---|---|
| CA-01 | Cart ID SHALL be stored in an HTTP-only cookie (`_medusa_cart_id`) |
| CA-02 | Cart SHALL be created lazily on first item addition |
| CA-03 | Line item quantity SHALL be updatable inline on the cart page |
| CA-04 | Line item removal SHALL be confirmed with optimistic UI update |
| CA-05 | Cart SHALL display line item totals, subtotal, shipping estimate, and grand total |
| CA-06 | Cart mutations SHALL use Next.js Server Actions |

---

### 2.4 Checkout Flow

#### Step Specification

| Step | ID | Component | Validation |
|---|---|---|---|
| Contact info | CH-01 | `checkout/email` | Valid email format |
| Shipping address | CH-02 | `checkout/shipping-address` | Required: first/last name, address, city, postal code, country |
| Delivery method | CH-03 | `checkout/shipping` | At least one option must be selected |
| Payment | CH-04 | `checkout/payment` | Stripe PaymentElement completes without error |
| Review | CH-05 | `checkout/review` | All previous steps valid |
| Confirmation | CH-06 | `order/confirmed` | Order ID present in URL |

#### Checkout Acceptance Criteria

- [ ] User can complete checkout as guest (no account required)
- [ ] User can complete checkout as authenticated customer
- [ ] Stripe payment element loads within 2 seconds
- [ ] Successful payment redirects to `/[countryCode]/order/[id]/confirmed`
- [ ] Failed payment surfaces Stripe error message without data loss
- [ ] Browser back navigation restores previous checkout step state

---

### 2.5 Authentication & Account

| ID | Requirement |
|---|---|
| AU-01 | Login SHALL use email/password via `sdk.auth.emailpass.authenticate()` |
| AU-02 | Session token SHALL be stored in HTTP-only cookie; never in `localStorage` |
| AU-03 | Account page SHALL use Next.js parallel routes to swap between login and dashboard |
| AU-04 | Dashboard SHALL display: profile details, saved addresses, order history |
| AU-05 | Order detail page SHALL be accessible from order history |
| AU-06 | Logout SHALL clear session cookie and redirect to `/[countryCode]/` |
| AU-07 | Order transfer SHALL support accept/decline via `/order/[id]/transfer/[token]` |

---

## 3. Page Specifications

| Page | Route | Required Data | Rendering |
|---|---|---|---|
| Homepage | `/` | Featured collections, hero content | SSR |
| Store | `/store` | Products, regions | SSR + Streaming |
| Product Detail | `/products/[handle]` | Product, variants, region | SSG + ISR |
| Collection | `/collections/[handle]` | Collection + products | SSR |
| Category | `/categories/[...slug]` | Category tree + products | SSR |
| Cart | `/cart` | Cart (from cookie) | CSR |
| Checkout | `/checkout` | Cart, shipping options, payment session | CSR |
| Account | `/account` | Customer (authenticated) | CSR |
| Order Confirmed | `/order/[id]/confirmed` | Order | SSR |

---

## 4. Component Contracts

### `<ProductCard>`

```typescript
interface ProductCardProps {
  product: {
    id: string
    title: string
    handle: string
    thumbnail: string | null
    variants: Array<{ id: string; calculated_price: CalculatedPrice }>
  }
  region: Region
}
```

### `<CartItem>`

```typescript
interface CartItemProps {
  item: LineItem
  currencyCode: string
  onQuantityChange: (itemId: string, quantity: number) => Promise<void>
  onRemove: (itemId: string) => Promise<void>
}
```

---

## 5. Data-Fetching Interface Specification

All data-fetching functions are located in `src/lib/data/` and are server-only.

```typescript
// Products
getProductByHandle(handle: string, regionId: string): Promise<HttpTypes.StoreProduct>
listProducts(params: { regionId: string; page?: number; limit?: number }): Promise<{ products: StoreProduct[]; count: number }>

// Cart
retrieveCart(cartId: string): Promise<StoreCart>
addToCart(cartId: string, variantId: string, quantity: number): Promise<StoreCart>
setCartShippingAddress(cartId: string, address: StoreCartAddress): Promise<StoreCart>

// Customer
getCustomer(): Promise<StoreCustomer | null>   // Returns null if unauthenticated
login(email: string, password: string): Promise<void>
logout(): Promise<void>

// Payment
initiatePaymentSession(cartId: string, providerId: string): Promise<StorePaymentSession>
placeOrder(cartId: string): Promise<StoreOrder>
```

---

## 6. Non-Functional Requirements

| ID | Category | Requirement | Target |
|---|---|---|---|
| NFR-01 | Performance | Largest Contentful Paint (product page) | < 2.5 s |
| NFR-02 | Performance | Time to Interactive (checkout page) | < 3.5 s |
| NFR-03 | SEO | Product pages indexed | Lighthouse SEO score ≥ 90 |
| NFR-04 | Accessibility | WCAG compliance | 2.1 AA |
| NFR-05 | Security | Customer JWT storage | HTTP-only cookie only |
| NFR-06 | Security | No API secrets in browser bundle | Build-time guard via `server-only` |
| NFR-07 | Resilience | Backend unavailable | Graceful error boundaries; no blank pages |

---

## 7. Environment Variable Specification

```env
NEXT_PUBLIC_MEDUSA_BACKEND_URL     # Required: Full URL to Medusa backend
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY # Required: Medusa store publishable key
NEXT_PUBLIC_BASE_URL               # Required: Canonical storefront URL
NEXT_PUBLIC_DEFAULT_REGION         # Optional: Default region code (e.g. "us")
NEXT_PUBLIC_STRIPE_KEY             # Required for Stripe: Stripe publishable key
```

---

## 8. Build Specification

| Command | Description | Output |
|---|---|---|
| `yarn dev` | Dev server on port 8000 with Turbopack | Hot-reloading dev server |
| `yarn build` | Production build | `.next/` directory |
| `yarn start` | Start production server | HTTP on port 8000 |
| `yarn lint` | ESLint + Next.js rules | Lint report |
| `yarn analyze` | Bundle analysis | `.next/analyze/*.html` |
