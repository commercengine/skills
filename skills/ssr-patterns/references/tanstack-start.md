# TanStack Start Patterns

## Package

```bash
npm install @commercengine/storefront
```

Import the TanStack Start wrapper from `@commercengine/storefront/tanstack-start`. Server-only code uses a separate entry at `@commercengine/storefront/tanstack-start/server`.

## Setup

### 1. Create Storefront Config (Universal)

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

### 2. Create Server Storefront (Server-Only Module)

```typescript
// lib/storefront.server.ts
import { createTanStackStartServerStorefront } from "@commercengine/storefront/tanstack-start/server";
import { storefrontConfig } from "./storefront";

const factory = createTanStackStartServerStorefront(storefrontConfig);

export function serverStorefront() {
  return factory.serverStorefront();
}
```

The `.server.ts` suffix ensures TanStack Start tree-shakes this out of the client bundle. The `createTanStackStartServerStorefront` import from `.../tanstack-start/server` includes a `server-only` guard.

### 3. StorefrontBootstrap Component

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

### 4. Root Layout

```tsx
// routes/__root.tsx
import { createRootRoute, Outlet } from "@tanstack/react-router";
import { StorefrontBootstrap } from "@/components/storefront-bootstrap";

export const Route = createRootRoute({
  component: RootComponent,
});

function RootComponent() {
  return (
    <html lang="en">
      <body>
        <StorefrontBootstrap />
        <Outlet />
      </body>
    </html>
  );
}
```

### 5. Environment Variables

```env
# .env
VITE_STORE_ID=your-store-id
VITE_API_KEY=your-api-key
```

## Accessor Rules

| Context | What to Use | Import From |
|---------|-------------|-------------|
| Route loaders | `storefront.publicStorefront()` | `lib/storefront` |
| Pre-rendering / head metadata | `storefront.publicStorefront()` | `lib/storefront` |
| Client-side components | `storefront.clientStorefront()` | `lib/storefront` |
| Client bootstrap | `storefront.bootstrap()` | `lib/storefront` |
| Public server functions | `storefront.publicStorefront()` | `lib/storefront` |
| Session-bound server functions | `serverStorefront()` | `lib/storefront.server` |

**Key distinction from Next.js**: TanStack Start's app-local `serverStorefront()` is **synchronous** (returns `SessionStorefrontSDK` directly), while Next.js's is async (`Promise<SessionStorefrontSDK>`). The server cookie adapter is wired internally via TanStack Start's `getCookie`/`setCookie`/`deleteCookie`.

## Key Patterns

### Route Loader (Public Reads)

Route loaders fetch data on the server before rendering. Use `publicStorefront()` for catalog reads:

```typescript
// routes/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { storefront } from "@/lib/storefront";

export const Route = createFileRoute("/")({
  loader: async () => {
    const sdk = storefront.publicStorefront();
    const { data, error } = await sdk.catalog.listSkus({ page: 1, limit: 20 });
    if (error) console.error("Failed to load products:", error.message);
    return { products: data?.skus ?? [] };
  },
  component: HomePage,
});

function HomePage() {
  const { products } = Route.useLoaderData();
  return (
    <div>
      {products.map((product) => (
        <div key={product.sku}>{product.variant_name || product.product_name}</div>
      ))}
    </div>
  );
}
```

### Server Functions (Public Reads)

For data fetching called from client hooks (e.g., React Query), wrap SDK calls in server functions:

```typescript
// lib/server-fns/catalog.ts
import { createServerFn } from "@tanstack/react-start";
import { storefront } from "@/lib/storefront";

export const fetchProducts = createServerFn({ method: "GET" })
  .inputValidator((d: { page?: number; limit?: number } | undefined) => d ?? {})
  .handler(async ({ data }) => {
    const sdk = storefront.publicStorefront();
    const { data: result, error } = await sdk.catalog.listProducts({
      page: data.page ?? 1,
      limit: data.limit ?? 20,
    });
    if (error) throw new Error(error.message);
    return result;
  });

export const fetchProductDetail = createServerFn({ method: "GET" })
  .inputValidator((d: string) => d)
  .handler(async ({ data: slug }) => {
    const sdk = storefront.publicStorefront();
    const { data: result, error } = await sdk.catalog.getProductDetail({
      product_id_or_slug: slug,
    });
    if (error) throw new Error(error.message);
    return result?.product ?? null;
  });

export const fetchCategories = createServerFn({ method: "GET" })
  .handler(async () => {
    const sdk = storefront.publicStorefront();
    const { data, error } = await sdk.catalog.listCategories();
    if (error) throw new Error(error.message);
    return data?.categories ?? [];
  });
```

### Server Functions (Session-Bound Mutations)

For operations that need user context (wishlist, cart via server), use the server storefront:

```typescript
// lib/server-fns/wishlist.ts
import { createServerFn } from "@tanstack/react-start";
import { serverStorefront } from "@/lib/storefront.server";

export const fetchWishlist = createServerFn({ method: "GET" })
  .handler(async () => {
    const sdk = serverStorefront();
    const { data, error } = await sdk.cart.getWishlist();
    if (error) throw new Error(error.message);
    return data ?? { products: [] };
  });

export const addToWishlist = createServerFn({ method: "POST" })
  .inputValidator((d: { productId: string; variantId?: string | null }) => d)
  .handler(async ({ data }) => {
    const sdk = serverStorefront();
    const { data: result, error } = await sdk.cart.addToWishlist(
      { product_id: data.productId, variant_id: data.variantId ?? null }
    );
    if (error) throw new Error(error.message);
    return result;
  });
```

Do not manually call `sdk.getUserId()` or pass `user_id` for ordinary wishlist/cart methods. The session-aware overloads resolve it automatically.

### Client-Side Hooks with Server Functions

Use React Query to call server functions from client components:

```typescript
// lib/hooks.ts
import { useQuery } from "@tanstack/react-query";
import { fetchProducts, fetchListSkus } from "@/lib/server-fns/catalog";

export function useProducts(options: { page?: number; limit?: number } = {}) {
  return useQuery({
    queryKey: ["products", options],
    queryFn: () => fetchProducts({ data: options }),
  });
}

export function useListSkus(options: { categoryId?: string; page?: number; limit?: number } = {}) {
  return useQuery({
    queryKey: ["listSkus", options],
    queryFn: () => fetchListSkus({
      data: {
        category_id: options.categoryId ? [options.categoryId] : undefined,
        page: options.page ?? 1,
        limit: options.limit ?? 20,
      },
    }),
    enabled: true,
  });
}
```

### Head Metadata / SEO

Use `head` in route definitions with `publicStorefront()`:

```typescript
export const Route = createFileRoute("/product/$slug")({
  loader: async ({ params }) => {
    const sdk = storefront.publicStorefront();
    const { data } = await sdk.catalog.getProductDetail({ product_id: params.slug });
    return { product: data?.product ?? null };
  },
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData?.product?.name ?? "Product Not Found" },
      {
        name: "description",
        content: loaderData?.product?.short_description ?? "",
      },
    ],
  }),
  component: ProductPage,
});
```

### Pre-rendering

Enable pre-rendering in `vite.config.ts`:

```typescript
import { tanstackStart } from "@tanstack/react-start/plugin";

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        enabled: true,
        crawlLinks: true,
        autoStaticPathsDiscovery: true,
        filter: ({ path }) =>
          !path.startsWith("/search") && !path.endsWith(".xml"),
      },
    }),
  ],
});
```

Pre-rendered routes automatically use `publicStorefront()` from route loaders. Dynamic routes (search, user-specific) should be excluded via the `filter`.

## Hosted Checkout + TanStack Start

```typescript
// lib/storefront.ts — add onTokensUpdated
export const storefrontConfig = {
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
  onTokensUpdated: (accessToken, refreshToken) => {
    if (typeof window !== "undefined") {
      void import("@commercengine/checkout").then(({ getCheckout }) => {
        getCheckout().updateTokens(accessToken, refreshToken);
      });
    }
  },
};
```

```tsx
// components/storefront-bootstrap.tsx
import { useEffect } from "react";
import { initCheckout } from "@commercengine/checkout";
import { destroyCheckout } from "@commercengine/checkout/react";
import { storefront } from "@/lib/storefront";

export function StorefrontBootstrap() {
  useEffect(() => {
    const init = async () => {
      await storefront.bootstrap();

      const sdk = storefront.clientStorefront();
      const accessToken = await sdk.getAccessToken();
      const refreshToken = await sdk.session.peekRefreshToken();

      initCheckout({
        storeId: import.meta.env.VITE_STORE_ID,
        apiKey: import.meta.env.VITE_API_KEY,
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
  return null;
}
```

## Cloudflare Workers Note

Cloudflare Workers/workerd doesn't send a `User-Agent` header by default. The CE API requires one. Patch global fetch on the server:

```typescript
// lib/storefront.ts (top of file)
if (typeof window === "undefined") {
  const originalFetch = globalThis.fetch;
  globalThis.fetch = (input: RequestInfo | URL, init?: RequestInit) => {
    const headers = new Headers(
      input instanceof Request ? input.headers : init?.headers
    );
    if (!headers.has("User-Agent")) {
      headers.set("User-Agent", "myapp/1.0");
    }
    if (input instanceof Request) {
      return originalFetch(new Request(input, { headers }), init);
    }
    return originalFetch(input, { ...init, headers });
  };
}
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Importing `storefront.server.ts` in client code | Server storefront must only be imported in server functions — use the `.server.ts` file convention |
| CRITICAL | Using `publicStorefront()` for session-bound operations | Use `serverStorefront()` from the server entry for auth/cart/wishlist |
| HIGH | Missing bootstrap in root layout | Mount `StorefrontBootstrap` in `__root.tsx` |
| HIGH | Using `clientStorefront()` inside a `createServerFn` handler | Server functions run on the server — use `serverStorefront()` |
| MEDIUM | Not excluding dynamic routes from pre-rendering | Add a `filter` in the prerender config to exclude search, user-specific, and API routes |
| MEDIUM | Missing User-Agent on Cloudflare Workers | Patch global fetch to inject User-Agent header |
