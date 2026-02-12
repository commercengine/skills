---
name: ce-nextjs-patterns
description: Advanced Next.js patterns for Commerce Engine - storefront() function, CookieTokenStorage for SSR, StorefrontSDKInitializer, Server Actions, SSG, and ISR.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Next.js Patterns

For basic setup, see `setup/`.

## Impact Levels

- **CRITICAL** - Breaking bugs, security holes
- **HIGH** - Common mistakes
- **MEDIUM** - Optimization

## References

| Reference | Impact |
|-----------|--------|
| `references/server-vs-client.md` | CRITICAL - `storefront(cookies())` vs `storefront()` |
| `references/token-management.md` | HIGH - Cookie-based token flow in Next.js |

## Mental Model

The `storefront()` function adapts to the execution context:

| Context | Usage | Token Storage |
|---------|-------|---------------|
| **Client Components** | `storefront()` | Browser cookies |
| **Server Components** | `storefront(cookies())` | Request cookies |
| **Server Actions** | `storefront(cookies())` | Request cookies (read + write) |
| **Root Layout** | `storefront({ isRootLayout: true })` | Memory fallback |
| **Build time (SSG)** | `storefront()` | Memory (no user context) |

## Setup

### 1. Install

```bash
npm install @commercengine/storefront-sdk-nextjs
```

### 2. Create Config

```typescript
// lib/storefront.ts
export { storefront } from "@commercengine/storefront-sdk-nextjs";
```

### 3. Root Layout

```tsx
// app/layout.tsx
import { StorefrontSDKInitializer } from "@commercengine/storefront-sdk-nextjs/client";
import { storefront } from "@/lib/storefront";

// Root Layout requires explicit flag — no request context available
const sdk = storefront({ isRootLayout: true });
const { data: storeConfig } = await sdk.store.getStoreConfig();

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <StorefrontSDKInitializer />
        <h1>Welcome to {storeConfig?.store_config?.brand.name}</h1>
        {children}
      </body>
    </html>
  );
}
```

### 4. Environment Variables

```env
# .env.local
NEXT_PUBLIC_STORE_ID=your-store-id
NEXT_PUBLIC_API_KEY=your-api-key
NEXT_BUILD_CACHE_TOKENS=true  # Faster builds with token caching
```

## Key Patterns

### Server Component (Data Fetching)

```typescript
// app/products/page.tsx
import { storefront } from "@/lib/storefront";
import { cookies } from "next/headers";

export default async function ProductsPage() {
  const sdk = storefront(cookies());
  const { data, error } = await sdk.catalog.listProducts({
    query: { page: 1, limit: 20 },
  });

  if (error) return <p>Error: {error.message}</p>;

  return (
    <div>
      {data.products.map((product) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

### Server Actions (Mutations)

```typescript
// app/actions.ts
"use server";

import { storefront } from "@/lib/storefront";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export async function loginWithEmail(email: string) {
  const sdk = storefront(cookies());

  const { data, error } = await sdk.auth.loginWithEmail({
    email,
    register_if_not_exists: true,
  });

  if (error) return { error: error.message };
  return { otp_token: data.content.otp_token, otp_action: data.content.otp_action };
}

export async function verifyOtp(otp: string, otpToken: string, otpAction: string) {
  const sdk = storefront(cookies());

  const { data, error } = await sdk.auth.verifyOtp({
    otp,
    otp_token: otpToken,
    otp_action: otpAction,
  });

  if (error) return { error: error.message };
  redirect("/account");
}

export async function addToCart(cartId: string, productId: string, variantId: string) {
  const sdk = storefront(cookies());

  const { data, error } = await sdk.cart.addCartItem(cartId, {
    product_id: productId,
    variant_id: variantId,
    quantity: 1,
  });

  if (error) return { error: error.message };
  return { cart: data };
}
```

### Static Site Generation (SSG)

```typescript
// app/products/[slug]/page.tsx
import { storefront } from "@/lib/storefront";

// Pre-render product pages at build time
export async function generateStaticParams() {
  const sdk = storefront(); // No cookies at build time
  const { data } = await sdk.catalog.listProducts({ query: { limit: 100 } });

  return (data?.products ?? []).map((product) => ({
    slug: product.slug,
  }));
}

export default async function ProductPage({ params }: { params: { slug: string } }) {
  const sdk = storefront(); // No cookies for static pages
  const { data, error } = await sdk.catalog.getProduct({
    product_id_or_slug: params.slug,
  });

  if (error) return <p>Product not found</p>;
  const product = data.product;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.selling_price}</p>
      {/* AddToCartButton is a Client Component */}
    </div>
  );
}
```

### Client Component

```tsx
"use client";

import { storefront } from "@/lib/storefront";

export function AddToCartButton({ productId, variantId }: Props) {
  async function handleClick() {
    const sdk = storefront(); // No cookies in client components
    const { data, error } = await sdk.cart.addCartItem(cartId, {
      product_id: productId,
      variant_id: variantId,
      quantity: 1,
    });
  }

  return <button onClick={handleClick}>Add to Cart</button>;
}
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Missing `cookies()` in Server Components | Use `storefront(cookies())` for user-specific data on the server |
| CRITICAL | Auth in Server Components instead of Actions | Auth endpoints that return tokens MUST be in Server Actions, not Server Components |
| HIGH | Missing `StorefrontSDKInitializer` | Required in root layout for automatic anonymous auth and session continuity |
| HIGH | Using `cookies()` in Client Components | Client Components use `storefront()` (no cookies) — tokens managed via browser cookies |
| MEDIUM | Slow builds | Set `NEXT_BUILD_CACHE_TOKENS=true` for token caching during SSG |
| MEDIUM | Root Layout missing `isRootLayout` flag | Root Layout runs outside request context — use `storefront({ isRootLayout: true })` |

## See Also

- `setup/` - Basic SDK installation
- `auth/` - Authentication flows
- `cart-checkout/` - Cart management

## Documentation

- **Next.js Integration**: https://www.commercengine.io/docs/sdk/nextjs-integration
- **Token Management**: https://www.commercengine.io/docs/sdk/token-management
- **LLM Reference**: https://llm-docs.commercengine.io/sdk/
