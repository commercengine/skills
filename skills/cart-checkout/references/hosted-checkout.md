# Hosted Checkout

Commerce Engine Hosted Checkout is a pre-built, embeddable checkout that runs inside an iframe. It handles the entire purchase flow — cart, authentication, addresses, payments, and order confirmation.

## Why Hosted Checkout?

- **Ship faster** — skip building cart UI, payment forms, and checkout logic
- **PCI compliant** — payment data never touches your servers
- **Always up to date** — new payment methods and features deployed automatically
- **Framework agnostic** — React, Vue, Svelte, Solid, or plain HTML

## Packages

| Package | Description |
|---------|-------------|
| `@commercengine/checkout` | Universal SDK with React, Vue, Svelte, and Solid bindings |
| `@commercengine/js` | Vanilla JS SDK, also available via CDN |

## Setup by Framework

### React

```bash
npm install @commercengine/checkout
```

```typescript
// Initialize once at app entry point
import { initCheckout } from "@commercengine/checkout/react";

initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  environment: "production",
});
```

```tsx
// Use in any component
import { useCheckout } from "@commercengine/checkout/react";

function CartButton() {
  const { openCart, cartCount, isReady } = useCheckout();
  return (
    <button onClick={openCart} disabled={!isReady}>
      Cart ({cartCount})
    </button>
  );
}
```

### Next.js

```tsx
// providers.tsx ("use client")
"use client";
import { initCheckout } from "@commercengine/checkout/react";

initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  environment: "production",
});

export function CheckoutProvider({ children }: { children: React.ReactNode }) {
  return <>{children}</>;
}

// app/layout.tsx
import { CheckoutProvider } from "./providers";

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <CheckoutProvider>{children}</CheckoutProvider>
      </body>
    </html>
  );
}
```

### Vue

```bash
npm install @commercengine/checkout
```

```vue
<script setup lang="ts">
import { useCheckout } from "@commercengine/checkout/vue";

const { openCart, openCheckout, cartCount, isReady, addToCart } = useCheckout();
</script>

<template>
  <button @click="openCart" :disabled="!isReady">
    Cart ({{ cartCount }})
  </button>
  <button @click="addToCart('product_abc', 'variant_xyz')">
    Add to Cart
  </button>
</template>
```

### Svelte

```svelte
<script>
  import { initCheckout } from "@commercengine/checkout/svelte";

  initCheckout({
    storeId: "store_xxx",
    apiKey: "ak_xxx",
    environment: "production",
  });
</script>

<slot />
```

### Solid

```tsx
import { initCheckout } from "@commercengine/checkout/solid";

if (typeof window !== "undefined") {
  initCheckout({
    storeId: "store_xxx",
    apiKey: "ak_xxx",
    environment: "production",
  });
}
```

### Vanilla JS (CDN)

```html
<script src="https://cdn.commercengine.com/v1.js"></script>
<script>
  const checkout = Commercengine.init({
    storeId: "store_xxx",
    apiKey: "ak_xxx",
    environment: "production",
  });

  document.getElementById("cart-btn").onclick = () => checkout.openCart();
</script>
```

## Events

| Event | Payload | Description |
|-------|---------|-------------|
| `ready` | — | Iframe loaded |
| `open` | — | Overlay visible |
| `close` | — | Overlay closed |
| `complete` | `{ id, orderNumber }` | Order placed |
| `cart:updated` | `{ count, total, currency }` | Cart changed |
| `auth:login` | `{ accessToken, refreshToken, user }` | User logged in |
| `auth:logout` | `{ accessToken, refreshToken, user }` | User logged out |
| `error` | `{ message }` | Error occurred |

```typescript
// Vanilla JS event handling
checkout.on("complete", (order) => {
  console.log("Order:", order.orderNumber);
});
```

## Syncing External Auth

If your app manages its own auth, sync tokens:

```typescript
const { updateTokens } = useCheckout();
updateTokens(accessToken, refreshToken);
```
