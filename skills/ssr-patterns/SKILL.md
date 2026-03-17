---
name: ce-ssr-patterns
description: SSR/meta-framework patterns for Commerce Engine using @commercengine/storefront. Covers Next.js, TanStack Start, Astro, and SvelteKit with publicStorefront(), clientStorefront(), serverStorefront(), bootstrap, server functions/actions, pre-rendering, and cookie-based token management.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "2.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# SSR / Meta-Framework Patterns

For basic installation, see `setup/`. For the rarer case where you need custom request-bound SSR bindings with `@commercengine/ssr-utils` (Nuxt or other unsupported frameworks), see `ssr/`.

This skill covers frameworks with **first-party Commerce Engine wrappers**: Next.js, TanStack Start, Astro, and SvelteKit. All ship as subpath exports of `@commercengine/storefront`.

## Impact Levels

- **CRITICAL** - Session continuity bugs, auth bugs
- **HIGH** - Common mistakes
- **MEDIUM** - Performance or maintainability issues

## References

| Reference | When to Use |
|-----------|-------------|
| `references/nextjs.md` | CRITICAL - Next.js setup, accessor rules, Server Actions, SSG |
| `references/tanstack-start.md` | CRITICAL - TanStack Start setup, server functions, route loaders, pre-rendering |
| `references/astro.md` | CRITICAL - Astro setup, browser bootstrap, server-only cookie access |
| `references/sveltekit.md` | CRITICAL - SvelteKit setup, load functions, hooks, form actions |
| `references/token-management.md` | HIGH - Token flow, bootstrap pattern, cookie configuration |

## Shared Mental Model

All first-party SSR wrappers follow the same three-accessor pattern (Next.js, TanStack Start, Astro, SvelteKit):

| Accessor | Environment | Purpose |
|----------|-------------|---------|
| `publicStorefront()` | Server or client | Build-safe public reads (catalog, categories, store config). API-key-backed, no session. |
| `clientStorefront()` | Client only | Browser-side session client. Throws if called on server. |
| `serverStorefront()` | Server only | Request-bound session with cookie-backed token storage. Next.js: `await storefront.serverStorefront()`. TanStack Start: `serverStorefront()` from server-only module. Astro: `serverStorefront(Astro.cookies)`. SvelteKit: `serverStorefront(cookies)` in load/hooks/actions. |

### Bootstrap

SSR apps **must** call `storefront.bootstrap()` on the client (root layout mount) before later `serverStorefront()` reads depend on cookie continuity. Without it, session continuity is not guaranteed. See each framework's reference for the exact bootstrap pattern.

### Pre-rendering for SEO & Initial Load

Use `publicStorefront()` in route loaders (TanStack Start) or `generateStaticParams` / Server Components (Next.js) to pre-render catalog pages. This gives you excellent SEO and fast initial page loads:

```typescript
// Route loader (TanStack Start) or Server Component (Next.js)
const sdk = storefront.publicStorefront();
const { data } = await sdk.catalog.listProducts({ limit: 100 });
```

`publicStorefront()` never creates sessions, never reads/writes cookies, and is safe for build-time, pre-render, and ISR contexts.

### Client-Side Fetching After Hydration

Once the app is hydrated, **client-side fetching is equally fast and much less complex**. There is no need to wrap SDK calls in server functions — the SDK talks to a public API with no secrets to protect. Use React Query (or any fetching library) directly:

```typescript
// Public reads — no session needed
const sdk = storefront.publicStorefront();
const { data } = await sdk.catalog.listProducts({ limit: 20 });

// Session-bound reads (cart, wishlist, account) — use client accessor
const sessionSdk = storefront.clientStorefront();
const { data: wishlist } = await sessionSdk.cart.getWishlist();
```

The session-aware overloads resolve `user_id` automatically — no manual ID passing needed.

## Framework Quick Reference

| Framework | Import | Factory | Server Entry | Server Accessor | Env Prefix | Reference |
|-----------|--------|---------|-------------|-----------------|------------|-----------|
| Next.js | `.../nextjs` | `createNextjsStorefront()` | Built-in (async) | `await storefront.serverStorefront()` | `NEXT_PUBLIC_` | `references/nextjs.md` |
| TanStack Start | `.../tanstack-start` | `createTanStackStartStorefront()` | `.../tanstack-start/server` | `serverStorefront()` | `VITE_` | `references/tanstack-start.md` |
| Astro | `.../astro` | `createAstroStorefront()` | `.../astro/server` | `serverStorefront(Astro.cookies)` | `PUBLIC_` | `references/astro.md` |
| SvelteKit | `.../sveltekit` | `createSvelteKitStorefront()` | `.../sveltekit/server` | `serverStorefront(cookies)` | `PUBLIC_` | `references/sveltekit.md` |

All imports above are subpaths of `@commercengine/storefront`. See the framework-specific reference for full setup, accessor rules, patterns, and examples.

## Hosted Checkout + SSR

When using Hosted Checkout with an SSR framework, the Storefront SDK remains the session owner. Wire `onTokensUpdated` in the storefront config so checkout stays in sync on the client side:

```typescript
// In storefront config (both frameworks)
onTokensUpdated: (accessToken, refreshToken) => {
  if (typeof window !== "undefined") {
    void import("@commercengine/checkout").then(({ getCheckout }) => {
      getCheckout().updateTokens(accessToken, refreshToken);
    });
  }
},
```

Then initialize checkout in the bootstrap/initializer component after `storefront.bootstrap()` completes. No provider wrapper needed — `initCheckout` is called once and `useCheckout()` works globally.

> **Callback signature difference:** The SDK's `onTokensUpdated` receives two separate arguments `(accessToken, refreshToken)`. Checkout's `onTokensUpdated` receives a single object `({ accessToken, refreshToken })`. Do not mix these up.

```tsx
useEffect(() => {
  const init = async () => {
    await storefront.bootstrap();

    const sdk = storefront.clientStorefront();
    const accessToken = await sdk.getAccessToken();
    const refreshToken = await sdk.session.peekRefreshToken();

    initCheckout({
      storeId: /* ... */,
      apiKey: /* ... */,
      authMode: "provided",
      accessToken: accessToken ?? undefined,
      refreshToken: refreshToken ?? undefined,
      onTokensUpdated: ({ accessToken, refreshToken }) => {
        void sdk.setTokens(accessToken, refreshToken);
      },
    });
  };
  void init();
  return () => destroyCheckout();
}, []);
```

## Common Pitfalls (All SSR Frameworks)

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `publicStorefront()` for cart, auth, customer, order, or payment flows | Use the session accessor (`serverStorefront()` on server, `clientStorefront()` on client) |
| CRITICAL | Using `clientStorefront()` on the server | Use the server accessor — `clientStorefront()` throws on the server |
| CRITICAL | Using deprecated `@commercengine/storefront-sdk-nextjs` | Migrate to `@commercengine/storefront/nextjs` with `createNextjsStorefront()` |
| HIGH | Missing bootstrap component in root layout | Mount a client component calling `storefront.bootstrap()` once in the root layout |
| HIGH | Bootstrapping anonymous auth in SSG/pre-render code | Use `publicStorefront()` instead — pre-render code must not create sessions |
| MEDIUM | Fetching session-aware data in the root layout | Move it into a request-aware boundary and use the server accessor |

## See Also

- `setup/` - Basic SDK installation and framework detection
- `ssr/` - Custom SSR bindings with `@commercengine/ssr-utils` (Nuxt and other unsupported frameworks)
- `auth/` - Authentication flows
- `cart-checkout/` - Cart and Hosted Checkout patterns

## Documentation

- **SDK Documentation**: https://www.commercengine.io/docs/sdk
- **LLM Reference**: https://llm-docs.commercengine.io/sdk/
