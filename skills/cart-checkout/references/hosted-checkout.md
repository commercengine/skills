# Hosted Checkout

Commerce Engine Hosted Checkout is a pre-built, embeddable checkout that runs inside an iframe. It handles the entire purchase flow — cart review, authentication, address collection, payments, and order confirmation.

## Why Hosted Checkout?

- **Ship faster** — skip building cart UI, payment forms, and checkout logic
- **PCI compliant** — payment data never touches your servers
- **Always up to date** — new payment methods and features deployed automatically
- **Framework agnostic** — React, Vue, Svelte, Solid, or plain HTML
- **Zero layout shift** — iframe loads in the background

## Packages

| Package | Description |
|---------|-------------|
| `@commercengine/checkout` | Framework bindings for React, Vue, Svelte, and Solid |
| `@commercengine/js` | Vanilla JS SDK, also available via CDN |

## Configuration Reference

### Init Options

**Credentials:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `storeId` | `string` | — | Your Commerce Engine Store ID. Required unless `url` is provided. |
| `apiKey` | `string` | — | Your Commerce Engine API Key. Required unless `url` is provided. |
| `environment` | `"production" \| "staging"` | `"production"` | Determines which checkout app URL is loaded. |
| `url` | `string` | — | Override the checkout URL. Useful for local development. |

**Appearance:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `theme` | `"light" \| "dark" \| "system"` | `"system"` | Theme preference for the checkout UI. |
| `appearance.zIndex` | `number` | `99999` | z-index for the checkout overlay container. |
| `drawerDirection.mobile` | `"bottom" \| "top" \| "left" \| "right" \| "modal"` | `"bottom"` | Drawer slide direction on mobile viewports (<=767px). |
| `drawerDirection.desktop` | `"bottom" \| "top" \| "left" \| "right" \| "modal"` | `"right"` | Drawer slide direction on desktop viewports (>767px). |

**Authentication:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `authMode` | `"managed" \| "provided"` | `"managed"` | `"managed"`: checkout handles its own auth. `"provided"`: your app manages tokens and syncs them to checkout. |
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

**Feature Flags:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `features.loyalty` | `boolean` | `true` | Show loyalty points redemption. |
| `features.coupons` | `boolean` | `true` | Show coupon code input. |
| `features.collectInStore` | `boolean` | `false` | Enable collect-in-store fulfillment option. |
| `features.freeShippingProgress` | `boolean` | `true` | Show free shipping progress bar. |
| `features.productRecommendations` | `boolean` | `true` | Show product recommendations in cart. |

### Callbacks

| Callback | Payload | Description |
|----------|---------|-------------|
| `onReady` | — | Checkout iframe is loaded and ready. |
| `onOpen` | — | Checkout overlay became visible. |
| `onClose` | — | Checkout overlay was closed. |
| `onComplete` | `{ id, orderNumber }` | Order was placed successfully. |
| `onCartUpdate` | `{ count, total, currency }` | Cart contents changed. |
| `onLogin` | `{ accessToken, refreshToken, user, loginMethod }` | User logged in. `loginMethod` is `"whatsapp"`, `"phone"`, or `"email"`. |
| `onLogout` | `{ accessToken, refreshToken, user }` | User logged out. |
| `onTokenRefresh` | `{ accessToken, refreshToken }` | Tokens were automatically refreshed. |
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
| `auth:login` | `{ accessToken, refreshToken, user, loginMethod }` | User logged in. |
| `auth:logout` | `{ accessToken, refreshToken, user }` | User logged out. |
| `auth:refresh` | `{ accessToken, refreshToken }` | Tokens refreshed. |
| `auth:session-error` | — | Session invalid, tokens cleared. |
| `error` | `{ message }` | Error occurred. |

Event listener methods: `checkout.on(event, handler)`, `checkout.once(event, handler)`, `checkout.off(event, handler)`.

## Setup by Framework

### React

```bash
npm install @commercengine/checkout
```

```tsx
// Root component — initialize in useEffect, cleanup on unmount
import { useEffect } from "react";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/react";

function Root() {
  useEffect(() => {
    initCheckout({
      storeId: import.meta.env.VITE_STORE_ID,
      apiKey: import.meta.env.VITE_API_KEY,
      environment: "production",
      onComplete: (order) => console.log("Order completed:", order.orderNumber),
    });
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

Uses the React binding in a `"use client"` provider:

```tsx
// providers.tsx ("use client")
"use client";
import { useEffect } from "react";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/react";

export function CheckoutProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    initCheckout({
      storeId: process.env.NEXT_PUBLIC_STORE_ID!,
      apiKey: process.env.NEXT_PUBLIC_API_KEY!,
      environment: "production",
    });
    return () => destroyCheckout();
  }, []);

  return <>{children}</>;
}
```

```tsx
// app/layout.tsx
import { CheckoutProvider } from "./providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <CheckoutProvider>{children}</CheckoutProvider>
      </body>
    </html>
  );
}
```

Then use `useCheckout()` in any client component.

### Vue

```bash
npm install @commercengine/checkout
```

```typescript
// main.ts — initialize before createApp(), cleanup on HMR
import { createApp } from "vue";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/vue";
import App from "./App.vue";

initCheckout({
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  environment: "production",
  onComplete: (order) => console.log("Order completed:", order.orderNumber),
});

const app = createApp(App);
app.mount("#app");

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

**Nuxt 3:** Create `plugins/checkout.client.ts` (`.client.ts` suffix ensures browser-only):

```typescript
import { initCheckout } from "@commercengine/checkout/vue";

export default defineNuxtPlugin(() => {
  initCheckout({
    storeId: "store_xxx",
    apiKey: "ak_xxx",
    environment: "production",
  });
});
```

### Svelte

```bash
npm install @commercengine/checkout
```

**SvelteKit** — initialize in `src/routes/+layout.svelte` with `browser` guard and cleanup:

```svelte
<script>
  import { onMount, onDestroy } from "svelte";
  import { browser } from "$app/environment";
  import { initCheckout, destroyCheckout } from "@commercengine/checkout/svelte";

  onMount(() => {
    if (!browser) return;
    initCheckout({
      storeId: import.meta.env.VITE_STORE_ID,
      apiKey: import.meta.env.VITE_API_KEY,
      environment: "production",
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
npm install @commercengine/checkout
```

```tsx
// Root component — initialize in onMount, cleanup in onCleanup
import { onMount, onCleanup } from "solid-js";
import { initCheckout, destroyCheckout } from "@commercengine/checkout/solid";

function Root() {
  onMount(() => {
    initCheckout({
      storeId: import.meta.env.VITE_STORE_ID,
      apiKey: import.meta.env.VITE_API_KEY,
      environment: "production",
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

**SolidStart:** Initialize in root layout with `onMount`/`onCleanup`.

### Vanilla JS (ES Module)

```bash
npm install @commercengine/checkout
```

```typescript
import { initCheckout, getCheckout, subscribeToCheckout } from "@commercengine/checkout";

// Initialize
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  environment: "production",
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

### Vanilla JS (CDN)

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

Everything in Commerce Engine is linked to the user. Anonymous auth guarantees a `user_id` before anything happens — cart creation, analytics, order attribution. **There must be exactly one source of truth for the auth session.** Two separate sessions (your app and checkout managing auth independently) will cause disjointed carts, broken server-side analytics, and corrupted order attribution.

### Choosing the Right Mode

| Mode | When to use | Who manages auth |
|------|-------------|------------------|
| `managed` (default) | Your app does **not** manage Commerce Engine auth | Checkout handles everything (anonymous tokens, OTP login, refresh) |
| `provided` (advanced) | Your app **already** manages Commerce Engine auth (via SDK or direct API calls) | Your app is the single source of truth — checkout receives tokens |

**The rule is simple**: If your app manages CE auth (whether via the Storefront SDK or direct API calls), you **must** use `provided` mode. Using `managed` mode when your app also manages auth creates two independent sessions — this is a critical data integrity issue, not a preference.

### Managed Mode (Default)

Checkout handles the entire auth lifecycle — anonymous tokens, OTP login, token refresh. No auth code needed in your app.

If your app needs to know about auth events happening inside checkout (e.g., to show the user's name in your header), use the auth callbacks:

```typescript
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  // authMode defaults to "managed"

  onLogin: ({ accessToken, refreshToken, user, loginMethod }) => {
    // User logged in inside checkout — sync to your app if needed
    setAppUser(user);
    setAppTokens(accessToken, refreshToken);
  },
  onLogout: () => {
    clearAppUser();
  },
  onTokenRefresh: ({ accessToken, refreshToken }) => {
    setAppTokens(accessToken, refreshToken);
  },
});
```

### Provided Mode (Advanced)

Your app is the single source of truth for auth. Checkout receives tokens from your app and never manages its own auth state.

**This mode requires precision.** You must sync tokens to checkout on every auth state change — login, logout, token refresh. A stale or missing token in checkout means a broken session.

```typescript
initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  authMode: "provided",
  accessToken: currentAccessToken,
  refreshToken: currentRefreshToken,
});
```

**Sync on every token change** — checkout will not be ready until tokens are provided, and will break silently if tokens go stale:

```typescript
const { updateTokens } = useCheckout();

// On login or token refresh in your app:
updateTokens(newAccessToken, newRefreshToken);

// On logout in your app — clears checkout session:
updateTokens("", "");
```

### Auth Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | App manages CE auth (SDK or API) + `managed` mode | Two independent sessions — breaks analytics, cart association, and order attribution. **Must** use `provided` mode. |
| CRITICAL | `provided` mode without syncing tokens | Checkout waits indefinitely for tokens. Must call `updateTokens()` after init and on every token change (login, logout, refresh). |
| CRITICAL | Not syncing logout in `provided` mode | User logs out of your app but remains logged in inside checkout — different users on the same session. Sync via `updateTokens("", "")`. |
| HIGH | Not syncing token refresh in `provided` mode | Your app refreshes tokens but checkout holds stale ones — API calls inside checkout fail or map to wrong session. |

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

| Environment | Checkout App URL |
|-------------|-----------------|
| `production` | `https://checkout.commercengine.com` |
| `staging` | `https://staging.checkout.commercengine.com` |

The SDK itself is hosted at `https://cdn.commercengine.com/v1.js` (single deployment). The `environment` option only changes which checkout app URL is loaded inside the iframe.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `authMode: "provided"` without syncing tokens | Must call `updateTokens()` whenever tokens change, or user sessions will break |
| HIGH | Calling `useCheckout()` before `initCheckout()` | `initCheckout()` must run first — call it at app entry point, outside components |
| HIGH | Not handling `onComplete` event | Always listen for order completion to show confirmation or redirect |
| MEDIUM | Missing `browser` guard in SSR frameworks | Use `typeof window !== "undefined"` (Solid) or `browser` (SvelteKit) guard |
| MEDIUM | Not using feature flags | Disable unused features (`loyalty`, `coupons`) for a cleaner checkout experience |
