# @dtc/storefront — arc42 Architecture Documentation

> Template version: 8.2 | Application: @dtc/storefront | Next.js: 15.5.18 | React: 19

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

`@dtc/storefront` is the customer-facing web application for the DTC platform. It must:

- Render product catalogue pages with high SEO performance
- Provide a frictionless cart and multi-step checkout experience
- Support per-region pricing, currency, and locale
- Authenticate customers and expose an account management dashboard
- Integrate with Medusa backend APIs exclusively via `@medusajs/js-sdk`

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **SEO Performance** | Product pages indexed within hours; Core Web Vitals in "Good" range |
| 2 | **Conversion** | Checkout completion rate unaffected by rendering delays |
| 3 | **Accessibility** | WCAG 2.1 AA compliance for all interactive components |
| 4 | **Maintainability** | New page types added without touching existing modules |
| 5 | **Security** | No sensitive tokens exposed to browser; auth via HTTP-only cookies |

### 1.3 Stakeholders

| Role | Expectation |
|---|---|
| Customer | Fast, reliable shopping experience across regions |
| Merchant | SEO-optimised pages; accurate inventory and pricing |
| Frontend Developer | Clear module boundaries; typed SDK; ergonomic data layer |

---

## 2. Architecture Constraints

- **Framework**: Next.js 15 App Router — no migration to `pages/` directory
- **React**: Version 19 with concurrent features and Server Components
- **Styling**: TailwindCSS only — no CSS modules, no styled-components
- **API**: Medusa JS SDK — no direct fetch() to backend outside of `src/lib/data/`
- **Auth**: JWT stored in HTTP-only cookie set by Next.js middleware / Server Actions
- **Node.js**: ≥ 18 for deployment; Turbopack for local development

---

## 3. System Scope and Context

### 3.1 Business Context

```
Browser / Search Bot
        │
        ▼
┌───────────────────────┐        ┌──────────────────────┐
│   @dtc/storefront     │───────▶│   @dtc/backend       │
│   (Next.js 15)        │◀───────│   (Medusa Server)    │
│                       │        └──────────────────────┘
│   Port: 8000          │
└───────────────────────┘
         │
         ▼
  ┌─────────────┐
  │   Stripe    │  (payment element, client-side JS)
  └─────────────┘
```

### 3.2 Technical Context

| Interface | Protocol | Used by |
|---|---|---|
| Medusa Store API | HTTPS REST | `src/lib/data/*` (server), SDK (client) |
| Stripe JS | Browser SDK | Checkout payment step |
| Next.js CDN / Edge | HTTP/2 | Image optimisation, static assets |

---

## 4. Solution Strategy

The storefront is a **thin rendering and routing layer** built on React Server Components. The key strategies are:

1. **Server-first data fetching** — all data-access functions in `src/lib/data/` are server-side; no API keys are exposed to the browser.
2. **Module-per-domain** — `src/modules/` groups components by commerce domain (products, cart, checkout, account…), each independently testable.
3. **Region-aware routing** — `middleware.ts` resolves country codes at the edge, enabling per-region pricing without client-side branching.
4. **Progressive enhancement** — critical paths (product detail, collection listing) work without JavaScript; interactive features (cart, checkout) hydrate on demand.

---

## 5. Building Block View

### Level 1 — System

```
@dtc/storefront
├── App Router (src/app/)        Next.js routing, layouts, page.tsx files
├── Feature Modules (src/modules/)  Domain-grouped components + templates
├── Data Layer (src/lib/data/)   Server-only SDK wrappers
├── SDK Client (src/lib/config.ts)  @medusajs/js-sdk singleton
├── Middleware (middleware.ts)   Edge-level region resolution
└── Styles (src/styles/)        TailwindCSS global configuration
```

### Level 2 — Feature Modules

| Module | Components | Templates |
|---|---|---|
| `products` | Product card, image gallery, variant selector | Product detail, product listing |
| `cart` | Cart item, cart drawer, empty state | Cart page |
| `checkout` | Address form, shipping selector, payment element, review | Checkout form, checkout summary |
| `account` | Login form, register form, profile form, address book | Dashboard, login |
| `order` | Order summary, line items | Order confirmed, order detail |
| `layout` | Header, footer, nav, country selector | — |
| `common` | Button, input, modal, breadcrumbs | — |

---

## 6. Runtime View

### Sequence: Product Page Load (SSR)

```
Browser          Next.js Server         Medusa Backend
   │                   │                      │
   │── GET /gb/products/my-shirt ─────────────▶│ (not shown — pure Next.js)
   │                   │── getProductByHandle() ─────────────────────────▶│
   │                   │◀── { product, variants, pricing } ───────────────│
   │                   │── getRegion("gb") ──────────────────────────────▶│
   │                   │◀── { region } ────────────────────────────────────│
   │                   │── render ProductTemplate (RSC) ────────────────  │
   │◀── HTML (full page, SEO-ready) ──────────│                           │
   │── JS hydration (client components only)  │                           │
```

### Sequence: Add to Cart (Server Action)

```
Browser          Server Action          Medusa Backend
   │                   │                      │
   │── click "Add to Cart" ───────────────────▶│
   │                   │── sdk.store.cart.lineItems.create() ────────────▶│
   │                   │◀── { cart } ──────────────────────────────────────│
   │                   │── revalidatePath("/gb/cart")                      │
   │◀── updated cart count badge ─────────────│                           │
```

---

## 7. Deployment View

```
┌───────────────────────────────────────────┐
│  Next.js Production Server  :8000         │
│                                           │
│  Edge Runtime  middleware.ts              │
│     → Region detection + redirect         │
│                                           │
│  Node.js Runtime                          │
│     → RSC rendering                       │
│     → Server Actions                      │
│     → API Route Handlers                  │
│                                           │
│  Static Assets  /_next/static/*           │
│     → Served via CDN / Next.js built-in   │
└───────────────────────────────────────────┘
```

---

## 8. Cross-Cutting Concepts

### 8.1 Security

- The Medusa publishable API key (`NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY`) is safe to expose — it is a read-only store-scoped key.
- Customer JWTs are stored in **HTTP-only cookies**, never in `localStorage`.
- All mutations go through Server Actions; no auth tokens pass through client fetch.

### 8.2 Caching

- Product pages use Next.js **`fetch` cache** with `revalidate` on the store region fetch.
- Cart data is never cached (always fresh from the server).
- `revalidateTag("products")` can be triggered from the backend via a webhook to invalidate stale PDPs.

### 8.3 Image Optimisation

All product images are rendered through Next.js `<Image>` with explicit `width`/`height` to prevent layout shift. The Medusa backend serves images via a configurable CDN URL.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-01 | App Router (not Pages Router) | RSC reduces client JS; better Suspense streaming |
| AD-02 | Server Actions for mutations | Eliminates client-side API key exposure; simplifies auth cookie handling |
| AD-03 | `[countryCode]` dynamic segment | Region-aware without JS; CDN-cacheable per region |
| AD-04 | `server-only` in `src/lib/data/` | Build-time guard against accidental client-side data access |
| AD-05 | Module-per-domain in `src/modules/` | Domain isolation; parallel team ownership |

---

## 10. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| React 19 breaking changes | Medium | Pin exact versions; review upgrade notes |
| Stripe Elements require client-side JS | Low | Checkout is already CSR — acceptable |
| `[countryCode]` segment complicates absolute URL generation | Medium | `NEXT_PUBLIC_BASE_URL` + utility helper wraps all URL generation |
| No automated E2E test suite yet | High | Add Playwright coverage for critical checkout path |
