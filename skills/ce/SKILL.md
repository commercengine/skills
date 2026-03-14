---
name: ce
description: Commerce Engine router. Use when the user asks about building a storefront, setting up the SDK, auth, products, cart, checkout, orders, webhooks, SSR, Next.js, or TanStack Start e-commerce patterns.
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Commerce Engine Skills Router

These skills cover B2C storefronts. B2B storefronts use the same SDK patterns plus customer-group-aware pricing and headers.

## Current SDK Mental Model

All frameworks install `@commercengine/storefront` — a unified package with subpath exports per framework:

- **SPA** (`@commercengine/storefront`): `createStorefront(...)` → `storefront.public()` / `storefront.session()`
- **Next.js** (`@commercengine/storefront/nextjs`): `createNextjsStorefront(...)` → `storefront.publicStorefront()` / `storefront.clientStorefront()` / `await storefront.serverStorefront()`
- **TanStack Start** (`@commercengine/storefront/tanstack-start`): `createTanStackStartStorefront(...)` → `storefront.publicStorefront()` / `storefront.clientStorefront()` + server entry for `serverStorefront()`

Core principle: **public reads** use the public accessor (API-key-backed, build-safe). **Live session flows** (auth, cart, orders) use the session/client/server accessor.

> `@commercengine/storefront-sdk-nextjs` is **deprecated**. Use `@commercengine/storefront/nextjs` instead.

## By Task

**Setting up the SDK** -> `ce-setup`
- Framework detection and SDK install
- Session storage selection
- Environment variables and one-config setup

**Authentication & login** -> `ce-auth`
- Anonymous session bootstrap
- OTP login, password login, password registration, password reset
- Account/profile flows

**Products & catalog** -> `ce-catalog`
- Products, categories, variants, search
- Reviews and recommendations

**Cart & checkout** -> `ce-cart-checkout`
- Hosted Checkout (recommended)
- Custom checkout (advanced)
- Cart CRUD, coupons, loyalty points, payments

**Orders & returns** -> `ce-orders`
- Order creation, order history, shipments, payments
- Cancellation and returns

**Webhooks & events** -> `ce-webhooks`
- Event handling, signature verification, async processing

**Next.js / TanStack Start patterns** -> `ce-ssr-patterns`
- `publicStorefront()` / `clientStorefront()` / `serverStorefront()`
- Bootstrap, Server Actions/functions, pre-rendering, token management
- Concrete references for Next.js and TanStack Start

**Custom SSR bindings** -> `ce-ssr`
- `@commercengine/ssr-utils` for frameworks without a first-party wrapper (SvelteKit, Nuxt, Astro)
- `ServerTokenStorage` and `CookieAdapter`
- Public build reads vs live request sessions

## Storefront Pages

Canonical pages for a CE storefront and the skills/methods each needs:

| Page | Example Route | Skills | Key SDK Methods |
|------|---------------|--------|-----------------|
| Home | `/` | catalog | `listProducts`, `listCategories`, recommendations |
| Product Listing | `/products`, `/categories/[slug]` | catalog | `searchProducts`, `listProducts` |
| Product Detail | `/products/[slug]` | catalog | `getProductDetail`, `listProductVariants`, `listProductReviews` |
| Cart | Hosted Checkout drawer or `/cart` | cart-checkout | `useCheckout().openCart()` or `sdk.cart.*` |
| Checkout | Hosted Checkout drawer or `/checkout` | cart-checkout | `useCheckout().openCheckout()` or custom checkout flow |
| Login | `/login` | auth | `loginWithEmail`, `loginWithPhone`, `verifyOtp`, `loginWithPassword` |
| Account | `/account` | auth | `getUserDetails`, `updateUserDetails`, `changePassword` |
| Orders | `/account/orders` | orders | `listOrders` |
| Order Detail | `/account/orders/[id]` | orders | `getOrderDetails`, `listOrderShipments`, `listOrderPayments` |

## Converting an Existing Project

1. **Install SDK** - Follow `ce-setup`. Install `@commercengine/storefront`.
2. **Replace public catalog reads first** - Move listing/detail/category pages to `publicStorefront()` / `public()`.
3. **Add session-aware flows** - Auth, cart, checkout, account, orders should use `clientStorefront()` / `serverStorefront()` / `session()`.
4. **Adopt Hosted Checkout or custom checkout** - Follow `ce-cart-checkout`.
5. **Add framework-specific SSR behavior if needed** - `ce-ssr-patterns` (Next.js / TanStack Start) or `ce-ssr` (custom bindings).

> Replace one data source at a time. Keep existing UI components where possible and swap the data layer first.

## Decision Tree

```
User Request
    │
    ├─ "Set up SDK" / "Add Commerce Engine"          → ce-setup
    ├─ "Login" / "Auth" / "OTP"                      → ce-auth
    ├─ "Products" / "Categories" / "Search"          → ce-catalog
    ├─ "Cart" / "Checkout" / "Payments"              → ce-cart-checkout
    ├─ "Orders" / "Returns" / "Shipments"            → ce-orders
    ├─ "Webhooks" / "Events" / "Sync"                → ce-webhooks
    ├─ "Next.js" / "Server Actions"                  → ce-ssr-patterns (references/nextjs.md)
    ├─ "TanStack Start" / "Server functions"         → ce-ssr-patterns (references/tanstack-start.md)
    └─ "SSR" / "Cookies" / "Custom binding"          → ce-ssr
```

## Quick Navigation

- `/ce-setup`
- `/ce-auth`
- `/ce-catalog`
- `/ce-cart-checkout`
- `/ce-orders`
- `/ce-webhooks`
- `/ce-ssr-patterns`
- `/ce-ssr`
