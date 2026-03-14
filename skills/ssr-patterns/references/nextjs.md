# Next.js Patterns

## Package

```bash
npm install @commercengine/storefront
```

Import the Next.js wrapper from `@commercengine/storefront/nextjs`.

> `@commercengine/storefront-sdk-nextjs` is **deprecated**. Migrate to `@commercengine/storefront/nextjs`.

## Setup

### 1. Create Config

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

`createNextjsStorefront()` requires explicit `storeId` and `apiKey` — it does not infer from environment variables.

### 2. StorefrontBootstrap Component

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

### 3. Root Layout

The root layout is a Server Component, but it can render Client Components as children — this is standard Next.js composition. `StorefrontBootstrap` is marked `"use client"`, so the layout just renders it without needing to be a Client Component itself. Place it as high in the tree as possible (root layout is ideal) so the session is established before any downstream components need it.

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

### 4. Environment Variables

```env
# .env.local
NEXT_PUBLIC_STORE_ID=your-store-id
NEXT_PUBLIC_API_KEY=your-api-key
```

## Accessor Rules

| Context | Call | Why |
|---------|------|-----|
| Root Layout | `storefront.publicStorefront()` | Root layouts should stay public |
| Build time / SSG / `generateMetadata` | `storefront.publicStorefront()` | No live user session |
| Public Server Component | `storefront.publicStorefront()` | Public read, no request session |
| Session-aware Server Component | `await storefront.serverStorefront()` | Auto-reads cookies via `next/headers` |
| Server Action / Route Handler | `await storefront.serverStorefront()` | Can read and write cookies |
| Client Component | `storefront.clientStorefront()` | Uses the browser-side session client |

`serverStorefront()` is **async** — it dynamically imports `next/headers` and uses React `cache()` to dedupe within a single request.

`clientStorefront()` **throws** if called on the server. `serverStorefront()` **throws** if called in the browser.

## Key Patterns

### Public Server Component

```typescript
// app/products/page.tsx
import { storefront } from "@/lib/storefront";

export default async function ProductsPage() {
  const sdk = storefront.publicStorefront();
  const { data, error } = await sdk.catalog.listProducts({ page: 1, limit: 20 });

  if (error) return <p>Error: {error.message}</p>;

  return (
    <div>
      {data?.products.map((product) => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

### Session-Aware Server Component

```typescript
// app/account/page.tsx
import { storefront } from "@/lib/storefront";

export default async function AccountPage() {
  const sdk = await storefront.serverStorefront();
  const { data, error } = await sdk.customer.getCustomer();

  if (error) return <p>Error: {error.message}</p>;
  return <div>Welcome, {data?.customer?.first_name}</div>;
}
```

### Server Actions (Mutations)

```typescript
// app/actions.ts
"use server";

import { storefront } from "@/lib/storefront";
import { redirect } from "next/navigation";

export async function loginWithEmail(email: string) {
  const sdk = await storefront.serverStorefront();

  const { data, error } = await sdk.auth.loginWithEmail({
    email,
    register_if_not_exists: true,
  });

  if (error) return { error: error.message };
  return { otp_token: data?.otp_token, otp_action: data?.otp_action };
}

export async function addToCartAction(
  cartId: string,
  productId: string,
  variantId: string | null,
) {
  const sdk = await storefront.serverStorefront();
  const { data, error } = await sdk.cart.addDeleteCartItem(
    { id: cartId },
    { product_id: productId, variant_id: variantId, quantity: 1 }
  );
  return { data: data?.cart, error: error?.message };
}
```

### Static Site Generation (SSG)

```typescript
// app/products/[slug]/page.tsx
import { storefront } from "@/lib/storefront";

export async function generateStaticParams() {
  const sdk = storefront.publicStorefront();
  const { data } = await sdk.catalog.listProducts({ limit: 100 });

  return (data?.products ?? []).map((product) => ({
    slug: product.slug || product.id,
  }));
}

export default async function ProductPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const sdk = storefront.publicStorefront();
  const { data, error } = await sdk.catalog.getProductDetail({
    product_id_or_slug: slug,
  });

  if (error || !data) return <p>Product not found</p>;
  return <h1>{data.product.name}</h1>;
}
```

### Client Component

```tsx
"use client";

import { storefront } from "@/lib/storefront";

export function AddToCartButton({
  cartId,
  productId,
  variantId,
}: {
  cartId: string;
  productId: string;
  variantId: string | null;
}) {
  async function handleClick() {
    const sdk = storefront.clientStorefront();
    await sdk.cart.addDeleteCartItem(
      { id: cartId },
      { product_id: productId, variant_id: variantId, quantity: 1 }
    );
  }

  return <button onClick={handleClick}>Add to Cart</button>;
}
```

### SEO Metadata

```typescript
import type { Metadata } from "next";
import { storefront } from "@/lib/storefront";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const sdk = storefront.publicStorefront();
  const { data } = await sdk.catalog.getProductDetail({
    product_id_or_slug: slug,
  });

  const product = data?.product;
  if (!product) return { title: "Product Not Found" };

  return {
    title: product.name,
    description: product.short_description ?? undefined,
  };
}
```

## Hosted Checkout + Next.js

**Step 1: Add `onTokensUpdated` to storefront config**

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

Then use `useCheckout()` in any client component — no provider needed:

```tsx
"use client";
import { useCheckout } from "@commercengine/checkout/react";

function Header() {
  const { cartCount, openCart, isReady } = useCheckout();
  return (
    <button onClick={openCart} disabled={!isReady}>Cart ({cartCount})</button>
  );
}
```

## Migration from `@commercengine/storefront-sdk-nextjs`

| Old | New |
|-----|-----|
| `@commercengine/storefront-sdk-nextjs` | `@commercengine/storefront/nextjs` |
| `createStorefront()` | `createNextjsStorefront({ storeId, apiKey, ... })` |
| `storefront.public()` | `storefront.publicStorefront()` |
| `storefront.session()` (client) | `storefront.clientStorefront()` |
| `storefront.session(await cookies())` | `await storefront.serverStorefront()` |
| `StorefrontSDKInitializer` | Custom `StorefrontBootstrap` component calling `storefront.bootstrap()` |
| Env var auto-inference | Explicit `storeId` and `apiKey` in config |

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `publicStorefront()` for cart/auth/order flows | Use `await serverStorefront()` on server or `clientStorefront()` on client |
| CRITICAL | Using `clientStorefront()` on the server | Use `await storefront.serverStorefront()` — `clientStorefront()` throws on server |
| HIGH | Missing `StorefrontBootstrap` | Mount a client component calling `storefront.bootstrap()` in the root layout |
| HIGH | Calling `serverStorefront()` in a Client Component | Client Components must use `storefront.clientStorefront()` |
| HIGH | Bootstrapping anonymous auth in SSG/root-layout code | Use `storefront.publicStorefront()` instead |
| MEDIUM | Session-aware data in the root layout | Move it into a nested boundary and use `await storefront.serverStorefront()` |
