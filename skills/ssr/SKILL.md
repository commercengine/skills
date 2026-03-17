---
name: ce-ssr
description: Build custom SSR bindings with @commercengine/ssr-utils for frameworks without a first-party Commerce Engine wrapper (Nuxt and other unsupported frameworks). CookieAdapter, ServerTokenStorage, and public vs session patterns.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "2.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Custom SSR Bindings (`@commercengine/ssr-utils`)

> **Prerequisite**: SDK initialized. See `setup/`.

## When to Use This Skill

Use `@commercengine/ssr-utils` only for frameworks that do **not** have a first-party Commerce Engine wrapper:

| Framework | What to Use |
|-----------|-------------|
| Next.js | `@commercengine/storefront/nextjs` → see `ssr-patterns/` |
| TanStack Start | `@commercengine/storefront/tanstack-start` → see `ssr-patterns/` |
| Astro | `@commercengine/storefront/astro` → see `ssr-patterns/` |
| SvelteKit | `@commercengine/storefront/sveltekit` → see `ssr-patterns/` |
| Nuxt | `@commercengine/ssr-utils` → **this skill** |

## Why This Matters

The SDK model splits public reads from live session flows:

- `PublicStorefrontSDK` / `storefront.public()` for public build/prerender reads
- `SessionStorefrontSDK` / `storefront.session(...)` for live anonymous or logged-in user flows

`@commercengine/ssr-utils` exists only for the second case. It provides cookie-backed request storage for building custom framework bindings.

## Quick Reference

| Concept | What It Does |
|---------|-------------|
| `CookieAdapter` | Normalizes framework cookie APIs into `{ get, set, delete }` |
| `ServerTokenStorage` | Implements `TokenStorage` using any `CookieAdapter` |
| `createCookieAdapter(...)` | Helper for frameworks whose cookie store already has compatible `get` / `set` / `delete` methods |

## Decision Tree

```
User Request: "Add SSR support" / "Cookie-based auth" / "Server-side rendering"
    │
    ├─ Next.js?
    │   └─ YES → Use @commercengine/storefront/nextjs → see ssr-patterns/
    │
    ├─ TanStack Start?
    │   └─ YES → Use @commercengine/storefront/tanstack-start → see ssr-patterns/
    │
    ├─ Astro?
    │   └─ YES → Use @commercengine/storefront/astro → see ssr-patterns/
    │
    ├─ SvelteKit?
    │   └─ YES → Use @commercengine/storefront/sveltekit → see ssr-patterns/
    │
    ├─ Public build/prerender read?
    │   └─ YES → Use PublicStorefrontSDK / storefront.public()
    │
    ├─ Live request with cookies? (Nuxt, etc.)
    │   └─ Create CookieAdapter → ServerTokenStorage → SessionStorefrontSDK
    │
    └─ Optional: wrap the base config in your own framework helper
```

## Architecture

```
Public render / prerender
  └─ PublicStorefrontSDK
     └─ API key only
     └─ No token bootstrap, refresh, or cookie writes

Live SSR request
  └─ SessionStorefrontSDK
     └─ tokenStorage: ServerTokenStorage(adapter)
     └─ Reads and writes request cookies
     └─ Can bootstrap anonymous auth and refresh tokens
```

## Installation

```bash
npm install @commercengine/storefront @commercengine/ssr-utils
```

## Key Patterns

### 1. Build the CookieAdapter

**Nuxt / h3**

```typescript
import { deleteCookie, getCookie, setCookie } from "h3";

const adapter = {
  get: (name: string) => getCookie(event, name) ?? null,
  set: (name: string, value: string, options?: Parameters<typeof setCookie>[3]) =>
    setCookie(event, name, value, options),
  delete: (name: string) => deleteCookie(event, name),
};
```

> **Note**: SvelteKit and Astro now have first-party wrappers (`@commercengine/storefront/sveltekit` and `@commercengine/storefront/astro`). Use those instead of building custom adapters. See `ssr-patterns/`.

### 2. Create `ServerTokenStorage`

```typescript
import { ServerTokenStorage } from "@commercengine/ssr-utils";

const tokenStorage = new ServerTokenStorage(adapter, {
  prefix: "myapp_",
  maxAge: 2592000,
  path: "/",
  sameSite: "lax",
});
```

### 3. Create the session SDK

```typescript
import { SessionStorefrontSDK } from "@commercengine/storefront";

const sdk = new SessionStorefrontSDK({
  storeId: "your-store-id",
  apiKey: "your-api-key",
  tokenStorage,
});
```

### 4. Public render SDK

For build-time or public SSR reads, do not go through `ServerTokenStorage` at all:

```typescript
import { PublicStorefrontSDK } from "@commercengine/storefront";

const publicSdk = new PublicStorefrontSDK({
  storeId: "your-store-id",
  apiKey: "your-api-key",
});

const { data } = await publicSdk.catalog.listProducts();
```

### 5. Session bootstrap

If a custom SSR binding wants to establish the session eagerly at the start of a live request, use the session SDK:

```typescript
await sdk.ensureAccessToken();
const { data } = await sdk.cart.getWishlist();
```

For ordinary session-aware SDK calls such as `sdk.cart.getWishlist()`, `sdk.cart.addToWishlist()`, `sdk.cart.getUserCart()`, or `sdk.customer.listAddresses()`, you do not need to call `ensureAccessToken()` first. The session middleware and overloads handle session creation and `user_id` / `customer_id` resolution automatically.

## Hosted Checkout Token Sync (SSR)

If Hosted Checkout is present, the Storefront SDK should remain the session owner. Wire `onTokensUpdated` so the checkout runtime stays in sync on the client side.

```typescript
const sdk = new SessionStorefrontSDK({
  storeId: "...",
  apiKey: "...",
  tokenStorage,
  onTokensUpdated: (accessToken, refreshToken) => {
    if (typeof window !== "undefined") {
      import("@commercengine/checkout").then(({ getCheckout }) => {
        getCheckout().updateTokens(accessToken, refreshToken);
      });
    }
  },
});
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `BrowserTokenStorage` in SSR code | Use `ServerTokenStorage` |
| CRITICAL | Using session SDK for build/prerender public pages | Use `PublicStorefrontSDK` or `storefront.public()` |
| HIGH | Using `ssr-utils` directly in Next.js, TanStack Start, Astro, or SvelteKit | Use the first-party wrappers: `@commercengine/storefront/nextjs`, `/tanstack-start`, `/astro`, or `/sveltekit` |
| HIGH | Client and server cookie formats drifting | Keep prefix, path, secure, sameSite, and encoding aligned |
| MEDIUM | Scattering `ensureAccessToken()` across feature code | If you want eager bootstrap, centralize it in one request bootstrap/helper instead of calling it before every cart/auth/customer method |

## See Also

- `setup/` - SDK installation and framework detection
- `ssr-patterns/` - First-party SSR patterns for Next.js, TanStack Start, Astro, and SvelteKit
- `auth/` - Authentication flows
- `cart-checkout/` - Hosted Checkout sync details

## Documentation

- **Token Management**: https://www.commercengine.io/docs/sdk/token-management
- **LLM Reference**: https://llm-docs.commercengine.io/sdk/
