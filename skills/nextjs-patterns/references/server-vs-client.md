# Server vs Client Components with Commerce Engine

## CRITICAL: Different `storefront()` usage per context

The `storefront()` function from `@commercengine/storefront-sdk-nextjs` adapts based on context. Using it incorrectly breaks user sessions or returns wrong data.

## Rules

| Context | Call | Why |
|---------|------|-----|
| Server Component | `storefront(cookies())` | Needs request cookies for user session |
| Server Action | `storefront(cookies())` | Can read AND write cookies (token persistence) |
| Client Component | `storefront()` | Accesses cookies via browser automatically |
| Root Layout | `storefront({ isRootLayout: true })` | No request context — falls back to memory |
| Build time (SSG) | `storefront()` | No user context, no cookies |

## Server Components — Data Fetching

```typescript
// app/account/page.tsx (Server Component)
import { storefront } from "@/lib/storefront";
import { cookies } from "next/headers";

export default async function AccountPage() {
  const sdk = storefront(cookies());

  // This uses the logged-in user's token from cookies
  const { data: userData } = await sdk.auth.getUserDetails({ id: userId });
  const { data: ordersData } = await sdk.order.listOrders({
    page: 1, limit: 10,
  });

  return <div>Welcome, {userData?.user?.first_name}</div>;
}
```

## Server Actions — Mutations

Auth endpoints that return tokens **MUST** be in Server Actions, not Server Components. Server Actions can write cookies, Server Components cannot.

```typescript
// app/actions.ts
"use server";

import { storefront } from "@/lib/storefront";
import { cookies } from "next/headers";

// Auth mutation — writes new tokens to cookies
export async function loginAction(email: string) {
  const sdk = storefront(cookies());
  const { data, error } = await sdk.auth.loginWithEmail({
    email,
    register_if_not_exists: true,
  });
  return { data, error: error?.message };
}

// Cart mutation — modifies cart state
export async function addToCartAction(cartId: string, productId: string, variantId: string | null) {
  const sdk = storefront(cookies());
  const { data, error } = await sdk.cart.addDeleteCartItem(
    { id: cartId },
    { product_id: productId, variant_id: variantId, quantity: 1 }
  );
  return { data: data?.cart, error: error?.message };
}
```

## Client Components — Interactive UI

```tsx
"use client";

import { storefront } from "@/lib/storefront";
import { useState, useEffect } from "react";

export function CartWidget() {
  const [cart, setCart] = useState(null);

  useEffect(() => {
    async function loadCart() {
      const sdk = storefront(); // No cookies needed — browser handles it
      const { data } = await sdk.cart.getUserCart({ user_id: userId });
      setCart(data?.cart);
    }
    loadCart();
  }, []);

  return <span>Cart: {cart?.cart_items_count ?? 0} items</span>;
}
```

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|-------------|-----|
| `storefront()` in Server Component | Returns anonymous data, not user-specific | Use `storefront(cookies())` |
| `storefront(cookies())` in Client Component | Build error — `cookies()` is server-only | Use `storefront()` |
| Auth in Server Component | Tokens can't be written to cookies | Move auth to Server Action |
| Missing `isRootLayout` in Root Layout | Error: no request context | Use `storefront({ isRootLayout: true })` |
