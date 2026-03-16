# Hosted Checkout

Commerce Engine Hosted Checkout is a pre-built, embeddable checkout that runs inside an iframe. It handles the entire purchase flow — cart review, authentication, address collection, payments, and order confirmation.

## Why Hosted Checkout?

- **Ship faster** — skip building cart UI, payment forms, and checkout logic
- **PCI compliant** — payment data never touches your servers
- **Always up to date** — new payment methods and features deployed automatically
- **Framework agnostic** — React, Vue, Svelte, Solid, or plain HTML
- **Store isolation by default** — each store loads from its own subdomain origin
- **Zero layout shift** — iframe loads in the background

## Packages

| Package | Description |
|---------|-------------|
| `@commercengine/checkout` | Framework bindings for React, Vue, Svelte, and Solid |
| `@commercengine/js` | Vanilla JS SDK, also available via CDN |

## Checkout Studio (No-Code Customization)

Use **Checkout Studio** for checkout customization:

- **URL:** https://studio.checkout.commercengine.io
- Configure appearance, features, login methods, drawer behavior, and payment settings
- Save once to publish changes without storefront code changes

**Assistant guidance:** If the user asks about checkout customization, always share this link first: https://studio.checkout.commercengine.io

## Configuration Reference

### Init Options

**Credentials:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `storeId` | `string` | — | Your Commerce Engine Store ID. Required unless `url` is provided. |
| `apiKey` | `string` | — | Your Commerce Engine API Key. Required unless `url` is provided. |
| `environment` | `"production" \| "staging"` | `"production"` | Determines which subdomain pattern is used: `https://{storeId}.checkout.commercengine.com` or `https://{storeId}.staging.checkout.commercengine.com`. |
| `url` | `string` | — | Override checkout host resolution (preview builds or custom hosted checkout domains). |

**Appearance:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `theme` | `"light" \| "dark" \| "system"` | `"system"` | Per-session theme override. Primary visual configuration is managed in Checkout Studio. |
| `appearance.zIndex` | `number` | `99999` | z-index for the checkout overlay container. |

**Authentication:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `authMode` | `"managed" \| "provided"` | `"managed"` | `"provided"` (recommended): your app and checkout sync tokens bidirectionally — **required** when using `@commercengine/storefront`. `"managed"`: checkout handles its own auth (standalone embeds on static HTML / no-code platforms only). |
| `accessToken` | `string` | — | Initial access token (JWT). Used with `authMode: "provided"`. |
| `refreshToken` | `string` | — | Initial refresh token. Used with `authMode: "provided"`. |

**Quick Buy:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `quickBuy` | `object` | — | Open checkout directly for a specific product. |
| `quickBuy.productId` | `string` | — | Product ID. Required when using `quickBuy`. |
| `quickBuy.variantId` | `string \| null` | — | Variant ID, or `null` for products without variants. |
| `quickBuy.quantity` | `number` | `1` | Quantity to add. |
| `sessionMode` | `"continue-existing" \| "force-new"` | `"continue-existing"` | Whether to continue the user's existing cart or start fresh. |
| `autoDetectQuickBuy` | `boolean` | `false` | Auto-detect quick buy params from the parent URL. |

**Managed in Checkout Studio (not SDK init options):**

- Feature flags (`loyalty`, `coupons`, `collectInStore`, `freeShippingProgress`, `productRecommendations`)
- Login configuration (enabled methods and account switching behavior)
- Drawer direction (mobile/desktop)
- Payment provider selection
- Brand appearance tokens (colors, typography, shape system, sizing)

### Callbacks

| Callback | Payload | Description |
|----------|---------|-------------|
| `onReady` | — | Checkout iframe is loaded and ready. |
| `onOpen` | — | Checkout overlay became visible. |
| `onClose` | — | Checkout overlay was closed. |
| `onComplete` | `{ id, orderNumber }` | Order was placed successfully. |
| `onCartUpdate` | `{ count, total, currency }` | Cart contents changed. |
| `onTokensUpdated` | `{ accessToken, refreshToken }` | Tokens changed (anonymous auth, login, logout, refresh). Use this for token sync. **Note:** Checkout's callback receives a single object `({ accessToken, refreshToken })`, while the SDK's `onTokensUpdated` receives two separate arguments `(accessToken, refreshToken)`. Do not mix up the signatures. |
| `onLogin` | `{ user, loginMethod }` | User logged in (semantic event). `loginMethod` is `"whatsapp"`, `"phone"`, or `"email"`. |
| `onLogout` | `{ user }` | User logged out (semantic event). |
| `onSessionError` | — | Session became invalid. All tokens are cleared. |
| `onError` | `{ message }` | A configuration or runtime error occurred. |

### Methods

| Method | Description |
|--------|-------------|
| `openCart()` | Open the cart drawer. |
| `openCheckout()` | Open the checkout directly, skipping the cart drawer. |
| `close()` | Close the checkout overlay. |
| `addToCart(productId, variantId, quantity?)` | Add an item. `variantId` can be `null`. `quantity` defaults to `1`. |
| `updateTokens(accessToken, refreshToken?)` | Sync auth tokens from your app into checkout. |
| `getCart()` | Returns `{ count, total, currency }`. Vanilla JS only. |
| `getTokens()` | Returns latest `{ accessToken, refreshToken }` seen from `auth:tokens-updated`, or `null` if not received yet. Vanilla JS only. |
| `destroy()` | Remove the iframe and clean up all listeners. Vanilla JS only. |
| `destroyCheckout()` | Cleanup function — removes iframe and listeners. Exported from all framework bindings. Call on unmount. |
| `getCheckout()` | Returns current checkout state for imperative access. Vanilla JS only. |
| `subscribeToCheckout(selector, callback)` | Subscribe to state changes with a selector function. Vanilla JS only. |

### Events (Vanilla JS)

For event-driven integration using `checkout.on()`:

| Event | Payload | Description |
|-------|---------|-------------|
| `ready` | — | Iframe loaded and ready. |
| `open` | — | Overlay became visible. |
| `close` | — | Overlay was closed. |
| `complete` | `{ id, orderNumber }` | Order placed. |
| `cart:updated` | `{ count, total, currency }` | Cart changed. |
| `auth:tokens-updated` | `{ accessToken, refreshToken }` | Tokens changed (anonymous auth, login, logout, refresh). |
| `auth:login` | `{ user, loginMethod }` | User logged in (semantic event). |
| `auth:logout` | `{ user }` | User logged out (semantic event). |
| `auth:session-error` | — | Session invalid, tokens cleared. |
| `error` | `{ message }` | Error occurred. |

Event listener methods: `checkout.on(event, handler)`, `checkout.once(event, handler)`, `checkout.off(event, handler)`.

## Setup by Framework

> **All framework examples below use `authMode: "provided"` with two-way token sync.** This is the required pattern for any app that uses the Storefront SDK. For standalone embeds on static HTML / no-code platforms without the SDK, see [Managed Mode](#managed-mode-standalone-embeds-only).

### SPA Storefront Config (Shared)

All SPA frameworks (React, Vue, Svelte, Solid) share this storefront config. It wires the SDK→checkout direction of the token sync. Place this in a shared module (e.g., `lib/storefront.ts`):

```typescript
// lib/storefront.ts
import {
  BrowserTokenStorage,
  createStorefront,
} from "@commercengine/storefront";
import { getCheckout } from "@commercengine/checkout";

const tokenStorage = new BrowserTokenStorage("myapp_");

export const storefront = createStorefront({
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  session: {
    tokenStorage,
    onTokensUpdated: (accessToken, refreshToken) => {
      getCheckout().updateTokens(accessToken, refreshToken);
    },
  },
});
```

> **SSR frameworks** (Next.js, TanStack Start, Nuxt, SvelteKit) use their own storefront factory — see the Next.js section below and `ssr-patterns/` for details.

### React

```bash
npm install @commercengine/checkout @commercengine/storefront
```

```tsx
// Root component — initialize storefront + checkout with two-way sync
import { useEffect } from "react";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/react";
import { storefront } from "./lib/storefront";

function Root() {
  useEffect(() => {
    const init = async () => {
      const sessionSdk = storefront.session();
      const accessToken = await sessionSdk.ensureAccessToken();
      const refreshToken = await sessionSdk.session.peekRefreshToken();

      initCheckout({
        storeId: import.meta.env.VITE_STORE_ID,
        apiKey: import.meta.env.VITE_API_KEY,
        authMode: "provided",
        accessToken: accessToken ?? undefined,
        refreshToken: refreshToken ?? undefined,
        onTokensUpdated: ({ accessToken, refreshToken }) => {
          void sessionSdk.setTokens(accessToken, refreshToken);
        },
        onComplete: (order) => console.log("Order completed:", order.orderNumber),
      });
    };
    void init();
    return () => destroyCheckout();
  }, []);

  return <App />;
}
```

```tsx
// Use in any component — destructure the hook
import { useCheckout } from "@commercengine/checkout/react";

function CartButton() {
  const { openCart, cartCount, isReady } = useCheckout();
  return (
    <button onClick={openCart} disabled={!isReady}>
      Cart ({cartCount})
    </button>
  );
}

function AddToCartButton({ productId, variantId }: Props) {
  const { addToCart } = useCheckout();
  return (
    <button onClick={() => addToCart(productId, variantId, 1)}>
      Add to Cart
    </button>
  );
}
```

**`useCheckout()` Hook API** (same for React, Vue, Solid):

| Property | Type | Description |
|----------|------|-------------|
| `isReady` | `boolean` | `true` when the checkout iframe is loaded |
| `isOpen` | `boolean` | `true` when the checkout overlay is visible |
| `cartCount` | `number` | Number of items in the cart |
| `cartTotal` | `number` | Cart subtotal |
| `cartCurrency` | `string` | Currency code (e.g. `"INR"`) |
| `isLoggedIn` | `boolean` | Whether a user is logged in |
| `user` | `UserInfo \| null` | Current user info (decoded from JWT) |
| `openCart()` | `function` | Open the cart drawer |
| `openCheckout()` | `function` | Open checkout directly (skips cart) |
| `close()` | `function` | Close the overlay |
| `addToCart(productId, variantId, quantity?)` | `function` | Add an item to the cart |
| `updateTokens(accessToken, refreshToken?)` | `function` | Sync auth tokens from your app |

### Next.js

Uses the React binding in a `"use client"` provider. Since Next.js storefronts use `createNextjsStorefront()`, Hosted Checkout should sync against the client storefront.

**Step 1: Configure the storefront with checkout sync**

```typescript
// lib/storefront.ts
import { Environment } from "@commercengine/storefront";
import { createNextjsStorefront } from "@commercengine/storefront/nextjs";

export const storefront = createNextjsStorefront({
  storeId: process.env.NEXT_PUBLIC_STORE_ID!,
  apiKey: process.env.NEXT_PUBLIC_API_KEY!,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
  onTokensUpdated: (accessToken, refreshToken) => {
    if (typeof window !== "undefined") {
      void import("@commercengine/checkout").then(({ getCheckout }) => {
        getCheckout().updateTokens(accessToken, refreshToken);
      });
    }
  },
});
```

**Step 2: Bootstrap + init checkout in a root client component**

No provider wrapper needed — `initCheckout` is called once at the root and `useCheckout()` works globally.

```tsx
// components/storefront-bootstrap.tsx
"use client";

import { useEffect } from "react";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/react";
import { storefront } from "@/lib/storefront";

export function StorefrontBootstrap() {
  useEffect(() => {
    async function init() {
      await storefront.bootstrap();

      const sdk = storefront.clientStorefront();
      const accessToken = await sdk.getAccessToken();
      const refreshToken = await sdk.session.peekRefreshToken();

      initCheckout({
        storeId: process.env.NEXT_PUBLIC_STORE_ID!,
        apiKey: process.env.NEXT_PUBLIC_API_KEY!,
        authMode: "provided",
        accessToken: accessToken ?? undefined,
        refreshToken: refreshToken ?? undefined,
        onTokensUpdated: ({ accessToken, refreshToken }) => {
          void sdk.setTokens(accessToken, refreshToken);
        },
      });
    }

    init();
    return () => destroyCheckout();
  }, []);

  return null;
}
```

**Step 3: Mount in root layout**

```tsx
// app/layout.tsx
import { StorefrontBootstrap } from "@/components/storefront-bootstrap";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <StorefrontBootstrap />
        {children}
      </body>
    </html>
  );
}
```

**How the sync works:**
1. `StorefrontBootstrap` calls `storefront.bootstrap()` to eagerly establish the browser session
2. The SDK's `onTokensUpdated` pushes every token change to checkout
3. Checkout's `onTokensUpdated` pushes token changes back into the client storefront
4. Returning visitors rehydrate from the existing session cookies

Then use `useCheckout()` in any client component — no provider needed.

### Vue

```bash
npm install @commercengine/checkout @commercengine/storefront
```

```typescript
// main.ts — initialize storefront + checkout with two-way sync
import { createApp } from "vue";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/vue";
import { storefront } from "./lib/storefront";
import App from "./App.vue";

async function bootstrap() {
  const sessionSdk = storefront.session();
  const accessToken = await sessionSdk.ensureAccessToken();
  const refreshToken = await sessionSdk.session.peekRefreshToken();

  initCheckout({
    storeId: import.meta.env.VITE_STORE_ID,
    apiKey: import.meta.env.VITE_API_KEY,
    authMode: "provided",
    accessToken: accessToken ?? undefined,
    refreshToken: refreshToken ?? undefined,
    onTokensUpdated: ({ accessToken, refreshToken }) => {
      void sessionSdk.setTokens(accessToken, refreshToken);
    },
    onComplete: (order) => console.log("Order completed:", order.orderNumber),
  });

  const app = createApp(App);
  app.mount("#app");
}

void bootstrap();

if (import.meta.hot) {
  import.meta.hot.dispose(() => destroyCheckout());
}
```

```vue
<!-- Use in any component — destructure the composable -->
<script setup lang="ts">
import { useCheckout } from "@commercengine/checkout/vue";

const { openCart, openCheckout, addToCart, cartCount, isReady } = useCheckout();
</script>

<template>
  <button @click="openCart" :disabled="!isReady">
    Cart ({{ cartCount }})
  </button>
</template>
```

**Nuxt 3:** Create `plugins/checkout.client.ts` (`.client.ts` suffix ensures browser-only). See `ssr/` for Nuxt storefront setup with `@commercengine/ssr-utils`:

```typescript
import { initCheckout } from "@commercengine/checkout/vue";
import { storefront } from "~/lib/storefront";

export default defineNuxtPlugin(async () => {
  const sessionSdk = storefront.session();
  const accessToken = await sessionSdk.ensureAccessToken();
  const refreshToken = await sessionSdk.session.peekRefreshToken();

  initCheckout({
    storeId: "store_xxx",
    apiKey: "ak_xxx",
    authMode: "provided",
    accessToken: accessToken ?? undefined,
    refreshToken: refreshToken ?? undefined,
    onTokensUpdated: ({ accessToken, refreshToken }) => {
      void sessionSdk.setTokens(accessToken, refreshToken);
    },
  });
});
```

### Svelte

```bash
npm install @commercengine/checkout @commercengine/storefront
```

**SvelteKit** — initialize in `src/routes/+layout.svelte` with `browser` guard and cleanup. See `ssr/` for SvelteKit storefront setup with `@commercengine/ssr-utils`:

```svelte
<script>
  import { onMount, onDestroy } from "svelte";
  import { browser } from "$app/environment";
  import { initCheckout, destroyCheckout } from "@commercengine/checkout/svelte";
  import { storefront } from "$lib/storefront";

  onMount(async () => {
    if (!browser) return;

    const sessionSdk = storefront.session();
    const accessToken = await sessionSdk.ensureAccessToken();
    const refreshToken = await sessionSdk.session.peekRefreshToken();

    initCheckout({
      storeId: import.meta.env.VITE_STORE_ID,
      apiKey: import.meta.env.VITE_API_KEY,
      authMode: "provided",
      accessToken: accessToken ?? undefined,
      refreshToken: refreshToken ?? undefined,
      onTokensUpdated: ({ accessToken, refreshToken }) => {
        void sessionSdk.setTokens(accessToken, refreshToken);
      },
      onComplete: (order) => console.log("Order completed:", order.orderNumber),
    });
  });

  onDestroy(() => {
    if (browser) destroyCheckout();
  });
</script>

<slot />
```

```svelte
<!-- Any component — use the checkout store with $ prefix -->
<script>
  import { checkout } from "@commercengine/checkout/svelte";
</script>

<button onclick={() => $checkout.openCart()} disabled={!$checkout.isReady}>
  Cart ({$checkout.cartCount})
</button>
```

Access state via `$checkout.property` (Svelte auto-subscription syntax). Fine-grained stores also available: `cartCount`, `cartTotal`, `isReady`, `isOpen`.

### Solid

```bash
npm install @commercengine/checkout @commercengine/storefront
```

```tsx
// Root component — initialize storefront + checkout with two-way sync
import { onMount, onCleanup } from "solid-js";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/solid";
import { storefront } from "./lib/storefront";

function Root() {
  onMount(async () => {
    const sessionSdk = storefront.session();
    const accessToken = await sessionSdk.ensureAccessToken();
    const refreshToken = await sessionSdk.session.peekRefreshToken();

    initCheckout({
      storeId: import.meta.env.VITE_STORE_ID,
      apiKey: import.meta.env.VITE_API_KEY,
      authMode: "provided",
      accessToken: accessToken ?? undefined,
      refreshToken: refreshToken ?? undefined,
      onTokensUpdated: ({ accessToken, refreshToken }) => {
        void sessionSdk.setTokens(accessToken, refreshToken);
      },
      onComplete: (order) => console.log("Order completed:", order.orderNumber),
    });
  });

  onCleanup(() => destroyCheckout());

  return <App />;
}
```

```tsx
// Use in any component — do NOT destructure (breaks reactivity)
import { useCheckout } from "@commercengine/checkout/solid";

function CartButton() {
  const checkout = useCheckout();
  return (
    <button onClick={() => checkout.openCart()} disabled={!checkout.isReady}>
      Cart ({checkout.cartCount})
    </button>
  );
}
```

**SolidStart:** Initialize in root layout with `onMount`/`onCleanup` using the same `authMode: "provided"` pattern.

### Vanilla JS (ES Module)

```bash
npm install @commercengine/checkout @commercengine/storefront
```

```typescript
import {
  BrowserTokenStorage,
  createStorefront,
} from "@commercengine/storefront";
import { initCheckout, getCheckout, subscribeToCheckout } from "@commercengine/checkout";

// Set up storefront with SDK→checkout token sync
const tokenStorage = new BrowserTokenStorage("myapp_");
const storefront = createStorefront({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  session: {
    tokenStorage,
    onTokensUpdated: (accessToken, refreshToken) => {
      getCheckout().updateTokens(accessToken, refreshToken);
    },
  },
});

// Bootstrap session and init checkout with two-way sync
const sessionSdk = storefront.session();
const accessToken = await sessionSdk.ensureAccessToken();
const refreshToken = await tokenStorage.getRefreshToken();

initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  authMode: "provided",
  accessToken: accessToken ?? undefined,
  refreshToken: refreshToken ?? undefined,
  onTokensUpdated: ({ accessToken, refreshToken }) => {
    void sessionSdk.setTokens(accessToken, refreshToken);
  },
  onComplete: (order) => console.log("Order placed:", order.orderNumber),
});

// Subscribe to state changes with selectors
subscribeToCheckout(
  (state) => state.isReady,
  (isReady) => {
    document.getElementById("cart-btn").disabled = !isReady;
  }
);

subscribeToCheckout(
  (state) => state.cartCount,
  (count) => {
    document.getElementById("cart-count").textContent = count.toString();
  }
);

// Imperative actions via getCheckout()
document.getElementById("cart-btn").onclick = () => getCheckout().openCart();
document.getElementById("add-btn").onclick = () => {
  getCheckout().addToCart("product_abc", null, 1);
};
```

### Vanilla JS (CDN) — Managed Mode for Static HTML

For static HTML pages, no-code platforms (Webflow, Framer), or sites that do **not** use the Storefront SDK, checkout runs in `managed` mode and handles auth entirely on its own:

```html
<script src="https://cdn.commercengine.com/v1.js" async></script>
<script>
  Commercengine.onLoad = async () => {
    const checkout = await Commercengine.init({
      storeId: "store_xxx",
      apiKey: "ak_xxx",
      environment: "production",
    });

    document.getElementById("cart-btn").addEventListener("click", () => {
      checkout.openCart();
    });

    checkout.on("cart:updated", (cart) => {
      document.getElementById("cart-count").textContent = cart.count;
    });

    checkout.on("complete", (order) => {
      console.log("Order placed:", order.orderNumber);
    });
  };
</script>
```

## Auth Mode Guide

Everything in Commerce Engine is linked to the live session. Anonymous bootstrap guarantees a `user_id` before cart creation, analytics, and order attribution. **There must be exactly one source of truth for the auth session.** Two separate sessions (your app and checkout managing auth independently) cause disjointed carts, broken server-side analytics, and corrupted order attribution.

### Choosing the Right Mode

| Mode | When to use | Who manages auth |
|------|-------------|------------------|
| `provided` (recommended) | Your app uses `@commercengine/storefront` or makes direct Commerce Engine API calls | Your app is the single source of truth — tokens sync bidirectionally with checkout |
| `managed` | Standalone embed on a static HTML / no-code platform (Webflow, Framer, etc.) where the Storefront SDK is **not** used | Checkout handles everything (anonymous tokens, OTP login, refresh) |

**The rule is simple**: If your app uses the `@commercengine/storefront` package at all, you **must** use `provided` mode. The Storefront SDK creates and manages its own session (tokens) for every API call — cart, customer, orders, analytics. If checkout simultaneously manages its own auth via `managed` mode, two independent sessions are created. This breaks cart association, server-side analytics, and order attribution. This applies whether or not your app has custom auth UI — the SDK's existence alone requires `provided` mode.

### Provided Mode (Recommended)

Your app is the single source of truth for auth. Checkout receives tokens from your app and syncs any internal token changes back.

**Why two-way sync?** Both the SDK and checkout can independently refresh near-expired tokens during API calls. The bidirectional sync ensures both sides always hold the same tokens regardless of which side triggers a refresh. Login and logout events inside checkout (OTP flow, account switching) also produce new tokens that must flow back to the SDK, and vice versa. Logout returns new tokens with reduced privileges — the `onTokensUpdated` callback handles this automatically in both directions, no manual token clearing needed.

This is the correct mode for:
- Any app using `@commercengine/storefront` (SPA or SSR)
- Any app making direct Commerce Engine API calls
- Any app with custom auth UI (login, account pages, etc.)

```typescript
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  authMode: "provided",
  accessToken: currentAccessToken,
  refreshToken: currentRefreshToken,
  onTokensUpdated: ({ accessToken, refreshToken }) => {
    // Sync checkout's token changes back to SDK
    void sessionSdk.setTokens(accessToken, refreshToken);
  },
});
```

**Two-way sync pattern (Storefront SDK ↔ Hosted Checkout):**

```typescript
import {
  BrowserTokenStorage,
  createStorefront,
} from "@commercengine/storefront";
import { getCheckout, initCheckout } from "@commercengine/checkout";

const tokenStorage = new BrowserTokenStorage("ce_");

const storefront = createStorefront({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  session: {
    tokenStorage,
    onTokensUpdated: (accessToken, refreshToken) => {
      getCheckout().updateTokens(accessToken, refreshToken);
    },
  },
});

const sessionSdk = storefront.session();
const accessToken = await sessionSdk.ensureAccessToken();
const refreshToken = await tokenStorage.getRefreshToken();

initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  authMode: "provided",
  accessToken: accessToken ?? undefined,
  refreshToken: refreshToken ?? undefined,
  onTokensUpdated: ({ accessToken, refreshToken }) => {
    void sessionSdk.setTokens(accessToken, refreshToken);
  },
  onLogin: ({ user, loginMethod }) => {
    updateUserUI(user, loginMethod);
  },
  onLogout: ({ user }) => {
    updateUserUI(user ?? null);
  },
});
```

### Managed Mode (Standalone Embeds Only)

Checkout handles the entire auth lifecycle — anonymous tokens, OTP login, token refresh. No auth code needed in the host page.

**Use managed mode only when:**
- The host page is static HTML with no Commerce Engine SDK (Webflow, Framer, Squarespace, or similar no-code platforms)
- Internal Commerce Engine applications where checkout is the only CE integration

If your app imports `@commercengine/storefront` at all — even for public catalog reads — **do not use managed mode**. It will create a second independent session.

```typescript
// Standalone embed on a static HTML page — no Storefront SDK present
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  // authMode defaults to "managed"
  onComplete: (order) => console.log("Order:", order.orderNumber),
});
```

### Auth Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | App uses `@commercengine/storefront` + `managed` mode | Two independent sessions — breaks analytics, cart association, and order attribution. **Must** use `provided` mode. |
| CRITICAL | `provided` mode without two-way sync | Must set up `onTokensUpdated` callbacks in both directions — SDK→checkout via `updateTokens()` and checkout→SDK via `setTokens()`. |
| CRITICAL | Not syncing tokens bidirectionally | Both sides can independently refresh tokens. Without two-way sync, one side holds stale tokens — API calls fail or map to the wrong session. |
| HIGH | Using `managed` mode in a framework app (React, Next.js, Vue, etc.) with Storefront SDK | Even without custom auth UI, the SDK's session management conflicts with checkout's. Use `provided` mode. |

## Quick Buy

Open checkout directly for a specific product, bypassing the cart drawer:

```typescript
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  quickBuy: {
    productId: "prod_abc",
    variantId: "var_xyz", // null for products without variants
    quantity: 1,
  },
});
```

## Environment URLs

| Environment | Checkout App URL Pattern |
|-------------|--------------------------|
| `production` | `https://{storeId}.checkout.commercengine.com` |
| `staging` | `https://{storeId}.staging.checkout.commercengine.com` |

The SDK itself is hosted at `https://cdn.commercengine.com/v1.js` (single deployment). The `environment` option selects the hostname pattern, and `storeId` is encoded in the subdomain.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `authMode: "provided"` without two-way sync | Must set up `onTokensUpdated` callbacks in both directions — SDK→checkout via `updateTokens()` and checkout→SDK via `setTokens()` |
| HIGH | Calling `useCheckout()` before `initCheckout()` | `initCheckout()` must run first — call it at app entry point, outside components |
| HIGH | Not handling `onComplete` event | Always listen for order completion to show confirmation or redirect |
| MEDIUM | Missing `browser` guard in SSR frameworks | Use `typeof window !== "undefined"` (Solid) or `browser` (SvelteKit) guard |
| MEDIUM | Trying to set feature flags in SDK init config | Manage feature flags in Checkout Studio (remote config), not in SDK init options |
