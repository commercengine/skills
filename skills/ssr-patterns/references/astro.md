# Astro Patterns

## Package

```bash
npm install @commercengine/storefront
```

Import the Astro wrapper from `@commercengine/storefront/astro`. Server-only code uses a separate entry at `@commercengine/storefront/astro/server`.

## Setup

### 1. Create Storefront Config (Shared)

```typescript
// src/lib/storefront-config.ts
import { Environment } from "@commercengine/storefront";
import type { AstroStorefrontConfig } from "@commercengine/storefront/astro";

export const storefrontConfig: AstroStorefrontConfig = {
  storeId: import.meta.env.PUBLIC_STORE_ID,
  apiKey: import.meta.env.PUBLIC_API_KEY,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
};
```

### 2. Create Client Storefront

```typescript
// src/lib/storefront.ts
import { createAstroStorefront } from "@commercengine/storefront/astro";
import { storefrontConfig } from "./storefront-config";

export const storefront = createAstroStorefront(storefrontConfig);
```

### 3. Create Server Storefront (Server-Only Module)

```typescript
// src/lib/server-storefront.ts
import { createAstroServerStorefront } from "@commercengine/storefront/astro/server";
import { storefrontConfig } from "./storefront-config";

export const serverStorefront = createAstroServerStorefront(storefrontConfig);
```

The `@commercengine/storefront/astro/server` entry is isolated from the client entry to prevent server code from leaking into browser bundles.

### 4. Bootstrap (Root Layout)

```astro
---
// src/layouts/Layout.astro
---

<html lang="en">
  <head><slot name="head" /></head>
  <body>
    <slot />

    <script type="module">
      import { storefront } from "../lib/storefront";

      const run = () => void storefront.bootstrap().catch(console.error);

      document.addEventListener("astro:page-load", run);
      run();
    </script>
  </body>
</html>
```

`astro:page-load` is important because Astro's `ClientRouter` runs bundled scripts once and then swaps pages on later navigations.

### 5. Environment Variables

```env
# .env
PUBLIC_STORE_ID=your-store-id
PUBLIC_API_KEY=your-api-key
```

## Accessor Rules

| Context | What to Use | Import From |
|---------|-------------|-------------|
| Frontmatter public reads (catalog, categories) | `storefront.publicStorefront()` | `src/lib/storefront` |
| Browser scripts / client islands | `storefront.clientStorefront()` | `src/lib/storefront` |
| Client bootstrap | `storefront.bootstrap()` | `src/lib/storefront` |
| SSR pages (`.astro` with `output: "server"`) | `serverStorefront.serverStorefront(Astro.cookies)` | `src/lib/server-storefront` |
| API routes (`+server.ts`) | `serverStorefront.serverStorefront(context.cookies)` | `src/lib/server-storefront` |
| Middleware | `serverStorefront.serverStorefront(context.cookies)` | `src/lib/server-storefront` |

**Key distinction**: Astro's `serverStorefront()` requires passing `Astro.cookies` (or `context.cookies`) at the call site because Astro has no global request context. This is synchronous — no `await` needed.

## Key Patterns

### Public Reads in Frontmatter

```astro
---
import { storefront } from "../lib/storefront";

const sdk = storefront.publicStorefront();
const { data } = await sdk.catalog.listCategories();
const categories = data?.categories ?? [];
---

<nav>
  {categories.map((category) => <a href={`/category/${category.slug}`}>{category.name}</a>)}
</nav>
```

### Session-Aware Server Page

```astro
---
// src/pages/account.astro (SSR)
import { serverStorefront } from "../lib/server-storefront";

const sdk = serverStorefront.serverStorefront(Astro.cookies);
const { data, error } = await sdk.customer.getCustomer();

if (error) return Astro.redirect("/login");
---

<h1>Welcome, {data?.customer?.first_name}</h1>
```

### API Route (Session-Bound)

```typescript
// src/pages/api/wishlist.ts
import type { APIRoute } from "astro";
import { serverStorefront } from "../../lib/server-storefront";

export const GET: APIRoute = async ({ cookies }) => {
  const sdk = serverStorefront.serverStorefront(cookies);
  const { data, error } = await sdk.cart.getWishlist();

  if (error) return new Response(JSON.stringify({ error: error.message }), { status: 500 });
  return new Response(JSON.stringify(data));
};

export const POST: APIRoute = async ({ cookies, request }) => {
  const body = await request.json();
  const sdk = serverStorefront.serverStorefront(cookies);
  const { data, error } = await sdk.cart.addToWishlist({
    product_id: body.productId,
    variant_id: body.variantId ?? null,
  });

  if (error) return new Response(JSON.stringify({ error: error.message }), { status: 500 });
  return new Response(JSON.stringify(data));
};
```

### Client-Side Fetching (After Hydration)

Once the app is hydrated, use the client accessor directly in scripts or framework islands:

```typescript
// In a <script> or framework component
import { storefront } from "../lib/storefront";

const sdk = storefront.publicStorefront();
const { data } = await sdk.catalog.listProducts({ limit: 20 });

// Session-bound (cart, wishlist, account)
const sessionSdk = storefront.clientStorefront();
const { data: wishlist } = await sessionSdk.cart.getWishlist();
```

### Middleware

```typescript
// src/middleware.ts
import { defineMiddleware } from "astro:middleware";
import { serverStorefront } from "./lib/server-storefront";

export const onRequest = defineMiddleware(async (context, next) => {
  const sdk = serverStorefront.serverStorefront(context.cookies);
  const userId = await sdk.session.peekUserId();

  if (context.url.pathname.startsWith("/account") && !userId) {
    return context.redirect("/login");
  }

  context.locals.userId = userId;
  return next();
});
```

## Hosted Checkout + Astro

```typescript
// src/lib/storefront-config.ts — add onTokensUpdated
export const storefrontConfig: AstroStorefrontConfig = {
  storeId: import.meta.env.PUBLIC_STORE_ID,
  apiKey: import.meta.env.PUBLIC_API_KEY,
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

Bootstrap checkout after the storefront session is established:

```astro
<script type="module">
  import { storefront } from "../lib/storefront";
  import { initCheckout } from "@commercengine/checkout";

  async function init() {
    await storefront.bootstrap();

    const sdk = storefront.clientStorefront();
    const accessToken = await sdk.getAccessToken();
    const refreshToken = await sdk.session.peekRefreshToken();

    initCheckout({
      storeId: import.meta.env.PUBLIC_STORE_ID,
      apiKey: import.meta.env.PUBLIC_API_KEY,
      authMode: "provided",
      accessToken: accessToken ?? undefined,
      refreshToken: refreshToken ?? undefined,
      onTokensUpdated: ({ accessToken, refreshToken }) => {
        void sdk.setTokens(accessToken, refreshToken);
      },
    });
  }

  document.addEventListener("astro:page-load", () => void init());
  void init();
</script>
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `publicStorefront()` for cart/auth/order flows | Use `serverStorefront(cookies)` on server or `clientStorefront()` in browser |
| CRITICAL | Importing `server-storefront.ts` in client scripts | Server storefront must only be used in `.astro` frontmatter, API routes, or middleware |
| HIGH | Missing bootstrap in root layout | Add bootstrap script with `astro:page-load` listener |
| HIGH | Forgetting to pass `Astro.cookies` to `serverStorefront()` | Always pass `Astro.cookies` or `context.cookies` — there is no global cookie access |
| MEDIUM | Using `ssr-utils` directly instead of the first-party wrapper | Use `@commercengine/storefront/astro` and `/astro/server` |
