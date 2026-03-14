---
name: ce-setup
description: Set up the Commerce Engine TypeScript SDK in any project. Framework detection, public vs session client selection, token storage choices, environment variables, and migration guidance.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.1.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Setting Up Commerce Engine

This skill sets up the latest Commerce Engine SDK surface.

## Current Mental Model

All frameworks install `@commercengine/storefront` — a unified package with subpath exports:

- **SPA**: `import { createStorefront } from "@commercengine/storefront"` → `storefront.public()` / `storefront.session()`
- **Next.js**: `import { createNextjsStorefront } from "@commercengine/storefront/nextjs"` → `storefront.publicStorefront()` / `storefront.clientStorefront()` / `await storefront.serverStorefront()`
- **TanStack Start**: `import { createTanStackStartStorefront } from "@commercengine/storefront/tanstack-start"` → `storefront.publicStorefront()` / `storefront.clientStorefront()` + separate server entry for `serverStorefront()`

The `@commercengine/storefront-sdk-nextjs` package is **deprecated**. Use `@commercengine/storefront/nextjs` instead.

## Quick Reference

| Step | Action |
|------|--------|
| 1. Detect framework | Check `package.json` and config files |
| 2. Install SDK | `@commercengine/storefront` (all frameworks) |
| 3. Initialize storefront | SPA: `createStorefront(...)`, Next.js: `createNextjsStorefront(...)`, TanStack Start: `createTanStackStartStorefront(...)` |
| 4. Set env vars | `VITE_STORE_ID` / `NEXT_PUBLIC_STORE_ID` and `VITE_API_KEY` / `NEXT_PUBLIC_API_KEY` |
| 5. Bootstrap session | SPA: call `sdk.ensureAccessToken()` once during startup if you want eager session setup, SSR frameworks: `storefront.bootstrap()` in a client component |
| 6. Use the right accessor | Public reads: `publicStorefront()` / `public()`, Session flows: `clientStorefront()` / `serverStorefront()` / `session()` |
| 7. Hosted Checkout (if used) | Run checkout in `authMode: "provided"` and sync tokens both ways |

## Canonical Setup (Storefront + Hosted Checkout)

If the storefront uses Hosted Checkout, this is the canonical setup:

- The Storefront SDK owns session state.
- Hosted Checkout runs in `authMode: "provided"`.
- Initialize checkout once at app startup, not per route or component render.
- Token sync is mandatory in both directions:
  - SDK -> checkout via `updateTokens(...)`
  - checkout -> SDK via `onTokensUpdated`

```typescript
import {
  BrowserTokenStorage,
  createStorefront,
} from "@commercengine/storefront";
import { getCheckout, initCheckout } from "@commercengine/checkout";

const tokenStorage = new BrowserTokenStorage("myapp_");

export const storefront = createStorefront({
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  session: {
    tokenStorage,
    // Note: in SPA (createStorefront), onTokensUpdated is nested inside `session`.
    // In SSR wrappers (createNextjsStorefront / createTanStackStartStorefront),
    // it is a top-level config property — do not nest it under `session` there.
    onTokensUpdated: (accessToken, refreshToken) => {
      getCheckout().updateTokens(accessToken, refreshToken);
    },
  },
});

const sessionSdk = storefront.session();
const accessToken = await sessionSdk.ensureAccessToken();
const refreshToken = await tokenStorage.getRefreshToken();

initCheckout({
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  authMode: "provided",
  accessToken: accessToken ?? undefined,
  refreshToken: refreshToken ?? undefined,
  onTokensUpdated: ({ accessToken, refreshToken }) => {
    sessionSdk.setTokens(accessToken, refreshToken);
  },
});
```

## Framework Detection

Check `package.json` and config files to identify the framework:

| Indicator | Framework | Import Path | Session Storage |
|-----------|-----------|-------------|-----------------|
| `next` in deps + `next.config.*` | Next.js | `@commercengine/storefront/nextjs` | Built-in (cookie-backed) |
| `@tanstack/react-start` in deps | TanStack Start | `@commercengine/storefront/tanstack-start` | Built-in (cookie-backed) |
| `@sveltejs/kit` in deps | SvelteKit | `@commercengine/storefront` + `@commercengine/ssr-utils` | `ServerTokenStorage` |
| `nuxt` in deps | Nuxt | `@commercengine/storefront` + `@commercengine/ssr-utils` | `ServerTokenStorage` |
| `astro` in deps + SSR adapter | Astro (SSR) | `@commercengine/storefront` + `@commercengine/ssr-utils` | `ServerTokenStorage` |
| `vite.config.*` + browser app | React / Vue / Svelte / Solid SPA | `@commercengine/storefront` | `BrowserTokenStorage` |
| `express` in deps | Express / Node.js | `@commercengine/storefront` | `MemoryTokenStorage` or custom `TokenStorage` |
| None of above | Vanilla JS | `@commercengine/storefront` | `BrowserTokenStorage` |

## Decision Tree

```
User Request: "Set up Commerce Engine" / "Add e-commerce"
    │
    ├─ Read package.json + config files
    │
    ├─ Next.js detected?
    │   ├─ YES → Install @commercengine/storefront
    │   │        → Use createNextjsStorefront() from @commercengine/storefront/nextjs
    │   │        → Root layout uses storefront.publicStorefront() + StorefrontBootstrap
    │   │        → See ce-ssr-patterns for request-bound usage
    │   └─ NO ↓
    │
    ├─ TanStack Start detected?
    │   ├─ YES → Install @commercengine/storefront
    │   │        → Use createTanStackStartStorefront() from @commercengine/storefront/tanstack-start
    │   │        → Server-only: createTanStackStartServerStorefront() from .../tanstack-start/server
    │   └─ NO ↓
    │
    ├─ Other SSR framework (SvelteKit, Nuxt, Astro)?
    │   ├─ YES → Install @commercengine/storefront + @commercengine/ssr-utils
    │   │        → Use ServerTokenStorage with CookieAdapter
    │   └─ NO  → Install @commercengine/storefront
    │            → Use BrowserTokenStorage (SPA) or MemoryTokenStorage (Node)
    │
    ├─ Using Hosted Checkout?
    │   ├─ YES → Install @commercengine/checkout
    │   │        → authMode: "provided"
    │   │        → Two-way token sync (SDK ↔ checkout)
    │   └─ NO → SDK-only setup
    │
    └─ Create one storefront factory and use the framework-appropriate accessors
```

## Setup by Framework

### Next.js (App Router)

```bash
npm install @commercengine/storefront
```

```typescript
// lib/storefront.ts
import { Environment } from "@commercengine/storefront";
import { createNextjsStorefront } from "@commercengine/storefront/nextjs";

export const storefront = createNextjsStorefront({
  storeId: process.env.NEXT_PUBLIC_STORE_ID!,
  apiKey: process.env.NEXT_PUBLIC_API_KEY!,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
});
```

```tsx
// components/storefront-bootstrap.tsx
"use client";

import { useEffect } from "react";
import { storefront } from "@/lib/storefront";

export function StorefrontBootstrap() {
  useEffect(() => {
    storefront.bootstrap().catch(console.error);
  }, []);
  return null;
}
```

The root layout is a Server Component, but it can render `StorefrontBootstrap` (a Client Component) as a child — this is standard Next.js composition. Place the bootstrap component as high in the tree as possible.

```tsx
// app/layout.tsx (Server Component — this is fine)
import { storefront } from "@/lib/storefront";
import { StorefrontBootstrap } from "@/components/storefront-bootstrap";

const { data: storeConfig } = await storefront.publicStorefront().store.getStoreConfig();

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <StorefrontBootstrap />
        <header>{storeConfig?.store_config?.brand.name}</header>
        {children}
      </body>
    </html>
  );
}
```

> For the full Next.js request model, see `ce-ssr-patterns`.
>
> **Using Hosted Checkout?** See `ce-cart-checkout` and `references/hosted-checkout.md` for the token sync pattern.

```env
# .env.local
NEXT_PUBLIC_STORE_ID=your-store-id
NEXT_PUBLIC_API_KEY=your-api-key
```

### TanStack Start

```bash
npm install @commercengine/storefront
```

```typescript
// lib/storefront.ts
import { Environment } from "@commercengine/storefront";
import { createTanStackStartStorefront } from "@commercengine/storefront/tanstack-start";

export const storefrontConfig = {
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
};

export const storefront = createTanStackStartStorefront(storefrontConfig);
```

```typescript
// lib/storefront.server.ts (server-only module)
import { createTanStackStartServerStorefront } from "@commercengine/storefront/tanstack-start/server";
import { storefrontConfig } from "./storefront";

const serverStorefrontFactory = createTanStackStartServerStorefront(storefrontConfig);

export function serverStorefront() {
  return serverStorefrontFactory.serverStorefront();
}
```

```tsx
// components/storefront-bootstrap.tsx
import { useEffect } from "react";
import { storefront } from "@/lib/storefront";

export function StorefrontBootstrap() {
  useEffect(() => {
    storefront.bootstrap().catch(console.error);
  }, []);
  return null;
}
```

Mount `StorefrontBootstrap` in the root layout (`__root.tsx`).

```env
# .env
VITE_STORE_ID=your-store-id
VITE_API_KEY=your-api-key
```

### React / Vue / Svelte / Solid (SPA)

```bash
npm install @commercengine/storefront
```

```typescript
// lib/storefront.ts
import {
  BrowserTokenStorage,
  Environment,
  createStorefront,
} from "@commercengine/storefront";

export const storefront = createStorefront({
  storeId: import.meta.env.VITE_STORE_ID,
  environment: import.meta.env.PROD ? Environment.Production : Environment.Staging,
  apiKey: import.meta.env.VITE_API_KEY,
  session: {
    tokenStorage: new BrowserTokenStorage("myapp_"),
  },
});
```

```typescript
// Public reads
const { data: products } = await storefront.public().catalog.listProducts();

// Live session flows
const sessionSdk = storefront.session();
await sessionSdk.ensureAccessToken();
```

```env
# .env
VITE_STORE_ID=your-store-id
VITE_API_KEY=your-api-key
```

### Node.js / Express

```bash
npm install @commercengine/storefront
```

```typescript
import {
  Environment,
  MemoryTokenStorage,
  createStorefront,
} from "@commercengine/storefront";

export const storefront = createStorefront({
  storeId: process.env.CE_STORE_ID!,
  environment: process.env.NODE_ENV === "production" ? Environment.Production : Environment.Staging,
  apiKey: process.env.CE_API_KEY!,
  session: {
    tokenStorage: new MemoryTokenStorage(),
  },
});
```

## Session Storage Guide

| Storage | Use Case | Persistence | SSR Safe |
|---------|----------|-------------|----------|
| `BrowserTokenStorage` | Browser SPAs | `localStorage` | No |
| `CookieTokenStorage` | Browser cookies / cross-tab cookie sync | Cookies | Browser only |
| `MemoryTokenStorage` | Server-side or temporary storage | In-memory | Yes |
| `ServerTokenStorage` | SSR frameworks with request cookies | Cookies via framework adapter | Yes |

Use `ServerTokenStorage` from `@commercengine/ssr-utils` for live SSR request flows. Do not use `CookieTokenStorage` as the primary SSR server storage layer.

## First API Call

Public reads can run immediately:

```typescript
const { data: products } = await storefront.public().catalog.listProducts();
```

If the app wants an eager live anonymous/logged-in session, do it once during startup:

```typescript
const sessionSdk = storefront.session();
await sessionSdk.ensureAccessToken();
const { data: wishlist } = await sessionSdk.cart.getWishlist();
```

Ordinary session-aware SDK methods such as `sdk.cart.getWishlist()`, `sdk.cart.addToWishlist()`, `sdk.cart.getUserCart()`, and `sdk.customer.listAddresses()` do not require a manual `ensureAccessToken()` call. The middleware and overloads handle session creation and ID resolution automatically. If your app wants eager bootstrap, call it once during startup instead of scattering `ensureAccessToken()` through feature code.

## Session Helpers

Most session SDK methods that need a `user_id` or `customer_id` have **parameterless overloads** that auto-resolve from the current session. You do not need to manually fetch IDs and pass them. For example:

```typescript
// Preferred — SDK auto-resolves user_id from session
const { data } = await sdk.cart.getUserCart();
const { data: wishlist } = await sdk.cart.getWishlist();
const { data: orders } = await sdk.order.listOrders();
const { data: addresses } = await sdk.customer.listAddresses();

// Only pass IDs explicitly when operating on behalf of a different user (admin scenarios)
const { data } = await sdk.cart.getUserCart({ user_id: "other_user_id" });
```

**Methods with parameterless overloads** (auto-resolve `user_id` or `customer_id`):
- **Cart**: `getUserCart()`, `deleteUserCart()`, `getWishlist()`, `addToWishlist()`, `removeFromWishlist()`
- **Orders**: `listOrders()`
- **Customer**: `listAddresses()`, `createAddress()`, `getAddress()`, `updateAddress()`, `deleteAddress()`, `getLoyaltyDetails()`, `listLoyaltyPointsActivity()`, `listCustomerReviews()`, `listSavedPaymentMethods()`, `listCustomerCards()`

**Auth client methods still require explicit IDs** — `getUserDetails({ id })`, `updateUserDetails({ id })`, etc. Use `sdk.getUserId()` to get the current user's ID for these calls.

### When You Still Need Session Helpers

| Helper | When to Use |
|--------|-------------|
| `ensureAccessToken()` | Call once during startup when you explicitly want to establish the session early |
| `getAccessToken()` | Read current token passively (passing to Hosted Checkout init) |
| `sdk.session.peekRefreshToken()` | Read current refresh token passively (passing to Hosted Checkout init) |
| `setTokens(accessToken, refreshToken?)` | Sync tokens from Hosted Checkout back into the SDK |
| `getUserId()` | Pass to auth client methods that require explicit `{ id }` |
| `getUserInfo()` | Read user info for UI display (name, email) without an API call — decoded from JWT |
| `isLoggedIn()` / `isAnonymous()` | Conditional UI rendering (show login button vs account menu) |

### `sdk.session` — Peek vs Ensure

The `session` property offers fine-grained access with two modes:

**Peek** — passive read, never creates sessions or refreshes tokens, returns `null` on failure:
- `sdk.session.peekAccessToken()` / `peekRefreshToken()` / `peekUserInfo()` / `peekUserId()` / `peekCustomerId()`

**Ensure** — may create an anonymous session or refresh expired tokens, throws on failure:
- `sdk.session.ensureAccessToken()` / `ensureUserInfo()` / `ensureUserId()` / `ensureCustomerId()`

Use **peek** when you want current state without side effects (e.g., passing tokens to Hosted Checkout init). Use **ensure** when you explicitly need a valid token or identity before any other session call. Most cart/order/customer methods already manage this internally.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CE_STORE_ID` / `NEXT_PUBLIC_STORE_ID` | Yes | Your store identifier |
| `CE_API_KEY` / `NEXT_PUBLIC_API_KEY` | Yes | Storefront API key |
| `CE_ENVIRONMENT` / `NEXT_PUBLIC_ENVIRONMENT` | No | `staging` or `production` |

## Default Headers

The SDK supports `defaultHeaders` in the config. This is useful for B2B storefronts with customer groups where pricing and promotions vary by group.

```typescript
const storefront = createStorefront({
  storeId: "...",
  apiKey: "...",
  defaultHeaders: {
    customer_group_id: "01JHS28V83KDWTRBXXJQRTEKA0",
  },
  session: {
    tokenStorage: new BrowserTokenStorage("myapp_"),
  },
});
```

Set `customer_group_id` after login from the user profile or auth response. Relevant catalog methods automatically receive it.

## Analytics

Analytics are server-side and automated. Commerce Engine collects e-commerce events per the Segment spec. Merchants route these events into Segment, Rudderstack, or similar tools via Admin integrations. Storefront code does not need to wire analytics manually.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| HIGH | Using `public()` for cart/auth/customer/order flows | Use `session()` for any live user or anonymous session work |
| HIGH | Using `CookieTokenStorage` as SSR server storage | Use `ServerTokenStorage` from `@commercengine/ssr-utils`, or use the built-in wrappers for Next.js / TanStack Start |
| HIGH | Skipping `StorefrontBootstrap` in SSR apps | Mount a client component calling `storefront.bootstrap()` once in the root layout |
| HIGH | Using deprecated `@commercengine/storefront-sdk-nextjs` | Migrate to `@commercengine/storefront/nextjs` with `createNextjsStorefront()` |
| MEDIUM | Bootstrapping anonymous auth in build/prerender code | Use `public()` instead |
| MEDIUM | Duplicating config across public and session clients | Prefer one `createStorefront(...)` factory |

## See Also

- `ssr-patterns/` - Next.js and TanStack Start `publicStorefront()` / `clientStorefront()` / `serverStorefront()` patterns
- `ssr/` - Custom SSR bindings with `@commercengine/ssr-utils` (SvelteKit, Nuxt, Astro)
- `auth/` - Authentication flows
- `cart-checkout/` - Hosted Checkout sync patterns

## Documentation

- **Core SDK**: https://www.commercengine.io/docs/sdk
- **Next.js Integration**: https://www.commercengine.io/docs/sdk/nextjs-integration
- **LLM Reference**: https://llm-docs.commercengine.io/sdk/
