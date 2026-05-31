# @dtc/storefront — Next.js Reference Storefront

> Next.js 15 · React 19 · Medusa JS SDK 2.15.3 · TailwindCSS

## Overview

`@dtc/storefront` is the customer-facing e-commerce frontend for the DTC platform. Built on **Next.js 15 App Router** with **React 19**, it delivers a high-performance shopping experience using React Server Components (RSC) for SEO-critical pages and client-side interactivity for the cart and checkout flow.

The storefront communicates with the Medusa backend exclusively through the **`@medusajs/js-sdk`** client, which is configured once in `src/lib/config.ts` and used throughout both server and client contexts.

---

## Project Structure

```
apps/storefront/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── layout.tsx                # Root HTML shell
│   │   └── [countryCode]/            # Region-aware routing
│   │       ├── (main)/               # Main storefront route group
│   │       │   ├── page.tsx          # Homepage
│   │       │   ├── store/            # All-products listing
│   │       │   ├── products/[handle] # Product detail page
│   │       │   ├── collections/[handle]
│   │       │   ├── categories/[...category]
│   │       │   ├── cart/             # Cart page
│   │       │   ├── account/          # Parallel route (login | dashboard)
│   │       │   │   ├── @login/       # Login slot
│   │       │   │   └── @dashboard/   # Dashboard slot (profile, addresses, orders)
│   │       │   └── order/[id]/       # Order confirmation & transfer
│   │       └── (checkout)/           # Checkout route group
│   │           └── checkout/         # Full-page checkout
│   ├── components/                   # Shared UI primitives
│   ├── lib/                          # Data access and utilities
│   │   ├── config.ts                 # Medusa SDK initialisation
│   │   ├── context/                  # React context providers
│   │   ├── data/                     # Server-side data-fetching functions
│   │   │   ├── cart.ts · products.ts · customer.ts
│   │   │   ├── payment.ts · fulfillment.ts · orders.ts
│   │   │   ├── regions.ts · collections.ts · categories.ts
│   │   │   └── cookies.ts · locales.ts · variants.ts
│   │   ├── hooks/                    # Client-side React hooks
│   │   └── util/                     # Pure utility functions
│   ├── modules/                      # Feature modules by domain
│   │   ├── account/                  # Authentication & account dashboard
│   │   ├── cart/                     # Cart display & item management
│   │   ├── checkout/                 # Multi-step checkout flow
│   │   ├── collections/              # Collection pages
│   │   ├── categories/               # Category pages
│   │   ├── common/                   # Shared UI components (nav, breadcrumbs)
│   │   ├── home/                     # Homepage components
│   │   ├── layout/                   # Header, footer, nav bar
│   │   ├── order/                    # Order confirmation & history
│   │   ├── products/                 # Product listing and detail
│   │   ├── shipping/                 # Shipping option display
│   │   ├── skeletons/                # Loading skeleton components
│   │   └── store/                    # Store listing page
│   ├── styles/                       # Global TailwindCSS styles
│   └── types/                        # Shared TypeScript types
└── middleware.ts                     # Next.js middleware (region redirect)
```

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- A running `@dtc/backend` instance (default: `http://localhost:9000`)

### Environment Variables

```env
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_...
NEXT_PUBLIC_BASE_URL=http://localhost:8000
NEXT_PUBLIC_DEFAULT_REGION=us
```

### Development

```bash
yarn install
yarn dev        # Starts Next.js on port 8000 with Turbopack
```

### Production Build

```bash
yarn build      # Static analysis + RSC compilation
yarn start      # Next.js production server on port 8000
```

---

## Routing & Region Awareness

All customer-facing routes are nested under `[countryCode]`, which is resolved by **`middleware.ts`** at the edge. The middleware inspects cookies and `Accept-Language` headers to redirect bare paths (`/products/shirt`) to region-prefixed paths (`/gb/products/shirt`), enabling per-region pricing and content.

---

## Key Pages & Rendering Strategy

| Route | Rendering | Description |
|---|---|---|
| `[countryCode]/` | SSR | Homepage hero & featured collections |
| `[countryCode]/store` | SSR + Streaming | Product listing with filters |
| `[countryCode]/products/[handle]` | SSR + `generateStaticParams` | Product detail (SEO-optimised) |
| `[countryCode]/collections/[handle]` | SSR | Collection product grid |
| `[countryCode]/cart` | CSR | Cart page (client cart context) |
| `[countryCode]/checkout` | CSR | Full-page checkout form |
| `[countryCode]/account` | CSR | Auth-gated account dashboard |
| `[countryCode]/order/[id]/confirmed` | SSR | Post-purchase confirmation |

---

## Medusa SDK Integration

```typescript
// src/lib/config.ts
import Medusa from "@medusajs/js-sdk"

export const sdk = new Medusa({
  baseUrl: process.env.NEXT_PUBLIC_MEDUSA_BACKEND_URL,
  publishableKey: process.env.NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY,
})
```

The SDK instance is augmented with an `x-medusa-locale` header injected from the current Next.js locale, ensuring region-aware pricing and translations are applied to every request.

---

## Key Dependencies

| Package | Version | Purpose |
|---|---|---|
| `next` | 15.5.18 | App framework |
| `react` / `react-dom` | 19.0.5 | UI runtime |
| `@medusajs/js-sdk` | 2.15.3 | Backend API client |
| `@medusajs/icons` | 2.15.3 | Icon set |
| `@medusajs/ui-preset` | 2.15.3 | TailwindCSS design tokens |
| `tailwindcss` | 3.x | Utility-first CSS |
| `@stripe/react-stripe-js` | 5.x | Stripe payment element |
| `@headlessui/react` | 2.x | Accessible UI primitives |
| `@radix-ui/react-accordion` | 1.x | Radix accordion primitive |
