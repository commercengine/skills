# SvelteKit Patterns

## Package

```bash
npm install @commercengine/storefront
```

Import the SvelteKit wrapper from `@commercengine/storefront/sveltekit`. Server-only code uses a separate entry at `@commercengine/storefront/sveltekit/server`.

## Setup

### 1. Create Storefront Config (Shared)

```typescript
// src/lib/storefront-config.ts
import { Environment } from "@commercengine/storefront";
import type { SvelteKitStorefrontConfig } from "@commercengine/storefront/sveltekit";
import { env } from "$env/static/public";

export const storefrontConfig: SvelteKitStorefrontConfig = {
  storeId: env.PUBLIC_STORE_ID,
  apiKey: env.PUBLIC_API_KEY,
  environment: Environment.Staging,
  tokenStorageOptions: { prefix: "myapp_" },
};
```

### 2. Create Client Storefront

```typescript
// src/lib/storefront.ts
import { createSvelteKitStorefront } from "@commercengine/storefront/sveltekit";
import { storefrontConfig } from "./storefront-config";

export const storefront = createSvelteKitStorefront(storefrontConfig);
```

### 3. Create Server Storefront (Server-Only Module)

```typescript
// src/lib/server/storefront.ts
import { createSvelteKitServerStorefront } from "@commercengine/storefront/sveltekit/server";
import { storefrontConfig } from "$lib/storefront-config";

export const serverStorefront = createSvelteKitServerStorefront(storefrontConfig);
```

Place this under `$lib/server/` so SvelteKit enforces that it is never imported into client code. Any file under `$lib/server/` is automatically server-only — SvelteKit throws a build error if client code imports from it.

### 4. Bootstrap (Root Layout)

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { onMount } from "svelte";
  import { storefront } from "$lib/storefront";

  onMount(() => {
    storefront.bootstrap().catch(console.error);
  });
</script>

<slot />
```

### 5. Environment Variables

```env
# .env
PUBLIC_STORE_ID=your-store-id
PUBLIC_API_KEY=your-api-key
```

SvelteKit uses the `PUBLIC_` prefix for client-exposed env vars (accessed via `$env/static/public`).

## Accessor Rules

| Context | What to Use | Import From |
|---------|-------------|-------------|
| Svelte component public reads | `storefront.publicStorefront()` | `$lib/storefront` |
| Svelte component session flows | `storefront.clientStorefront()` | `$lib/storefront` |
| Client bootstrap | `storefront.bootstrap()` | `$lib/storefront` |
| Server load functions (`+page.server.ts`, `+layout.server.ts`) | `serverStorefront.serverStorefront(cookies)` | `$lib/server/storefront` |
| Server hooks (`hooks.server.ts`) | `serverStorefront.serverStorefront(event.cookies)` | `$lib/server/storefront` |
| Form actions | `serverStorefront.serverStorefront(cookies)` | `$lib/server/storefront` |
| API routes (`+server.ts`) | `serverStorefront.serverStorefront(cookies)` | `$lib/server/storefront` |

**Key distinction**: SvelteKit's `serverStorefront()` requires passing `cookies` (from `event.cookies` or destructured) at the call site because SvelteKit has no global request context. This is synchronous — no `await` needed.

`clientStorefront()` **throws** if called on the server.

## Key Patterns

### Public Reads (Universal Load)

For public catalog data that doesn't need a session, use `publicStorefront()` in universal load functions. These run on both server and client:

```typescript
// src/routes/+page.ts
import { storefront } from "$lib/storefront";

export async function load() {
  const sdk = storefront.publicStorefront();
  const { data, error } = await sdk.catalog.listProducts({ page: 1, limit: 20 });

  if (error) return { products: [], error: error.message };
  return { products: data?.products ?? [] };
}
```

### Session-Aware Server Load

```typescript
// src/routes/account/+page.server.ts
import { serverStorefront } from "$lib/server/storefront";
import { redirect } from "@sveltejs/kit";

export async function load({ cookies }) {
  const sdk = serverStorefront.serverStorefront(cookies);
  const { data, error } = await sdk.customer.getCustomer();

  if (error) redirect(302, "/login");
  return { customer: data?.customer };
}
```

### Layout Server Load

```typescript
// src/routes/+layout.server.ts
import { serverStorefront } from "$lib/server/storefront";

export async function load({ cookies }) {
  const sdk = serverStorefront.serverStorefront(cookies);
  const userId = await sdk.session.peekUserId();

  return { isLoggedIn: !!userId };
}
```

### Form Actions (Mutations)

```typescript
// src/routes/cart/+page.server.ts
import { serverStorefront } from "$lib/server/storefront";
import { fail } from "@sveltejs/kit";

export const actions = {
  addToCart: async ({ cookies, request }) => {
    const formData = await request.formData();
    const productId = formData.get("productId") as string;
    const variantId = formData.get("variantId") as string | null;
    const cartId = formData.get("cartId") as string;

    const sdk = serverStorefront.serverStorefront(cookies);
    const { data, error } = await sdk.cart.addDeleteCartItem(
      { id: cartId },
      { product_id: productId, variant_id: variantId, quantity: 1 }
    );

    if (error) return fail(400, { error: error.message });
    return { cart: data?.cart };
  },

  addToWishlist: async ({ cookies, request }) => {
    const formData = await request.formData();
    const productId = formData.get("productId") as string;

    const sdk = serverStorefront.serverStorefront(cookies);
    const { data, error } = await sdk.cart.addToWishlist({
      product_id: productId,
      variant_id: null,
    });

    if (error) return fail(400, { error: error.message });
    return { wishlist: data };
  },
};
```

### API Routes

```typescript
// src/routes/api/wishlist/+server.ts
import { serverStorefront } from "$lib/server/storefront";
import { json, error } from "@sveltejs/kit";

export async function GET({ cookies }) {
  const sdk = serverStorefront.serverStorefront(cookies);
  const { data, error: apiError } = await sdk.cart.getWishlist();

  if (apiError) error(500, apiError.message);
  return json(data);
}

export async function POST({ cookies, request }) {
  const body = await request.json();
  const sdk = serverStorefront.serverStorefront(cookies);
  const { data, error: apiError } = await sdk.cart.addToWishlist({
    product_id: body.productId,
    variant_id: body.variantId ?? null,
  });

  if (apiError) error(500, apiError.message);
  return json(data);
}
```

### Server Hooks

```typescript
// src/hooks.server.ts
import { serverStorefront } from "$lib/server/storefront";
import type { Handle } from "@sveltejs/kit";

export const handle: Handle = async ({ event, resolve }) => {
  const sdk = serverStorefront.serverStorefront(event.cookies);
  const userId = await sdk.session.peekUserId();

  event.locals.userId = userId;

  if (event.url.pathname.startsWith("/account") && !userId) {
    return new Response(null, { status: 302, headers: { Location: "/login" } });
  }

  return resolve(event);
};
```

To make `event.locals.userId` type-safe, declare it in `app.d.ts`:

```typescript
// src/app.d.ts
declare global {
  namespace App {
    interface Locals {
      userId: string | null;
    }
  }
}

export {};
```

### Client-Side Fetching (After Hydration)

In Svelte components, use the client accessors directly:

```svelte
<script>
  import { onMount } from "svelte";
  import { storefront } from "$lib/storefront";

  let products = [];

  onMount(async () => {
    const sdk = storefront.publicStorefront();
    const { data } = await sdk.catalog.listProducts({ limit: 20 });
    products = data?.products ?? [];
  });
</script>

{#each products as product}
  <div>{product.name}</div>
{/each}
```

Session-bound operations:

```svelte
<script>
  import { storefront } from "$lib/storefront";

  async function addToWishlist(productId: string) {
    const sdk = storefront.clientStorefront();
    await sdk.cart.addToWishlist({
      product_id: productId,
      variant_id: null,
    });
  }
</script>
```

### Static Prerendering

For prerendered pages, use `publicStorefront()` in a universal load function:

```typescript
// src/routes/products/[slug]/+page.ts
import { storefront } from "$lib/storefront";

export const prerender = true;

export async function load({ params }) {
  const sdk = storefront.publicStorefront();
  const { data, error } = await sdk.catalog.getProductDetail({
    product_id_or_slug: params.slug,
  });

  if (error || !data) return { product: null };
  return { product: data.product };
}
```

`publicStorefront()` never creates sessions, never reads/writes cookies, and is safe for prerender and build-time contexts.

## Hosted Checkout + SvelteKit

```typescript
// src/lib/storefront-config.ts — add onTokensUpdated
export const storefrontConfig: SvelteKitStorefrontConfig = {
  storeId: env.PUBLIC_STORE_ID,
  apiKey: env.PUBLIC_API_KEY,
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

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { onMount, onDestroy } from "svelte";
  import { storefront } from "$lib/storefront";

  onMount(async () => {
    await storefront.bootstrap();

    const sdk = storefront.clientStorefront();
    const accessToken = await sdk.getAccessToken();
    const refreshToken = await sdk.session.peekRefreshToken();

    const { initCheckout } = await import("@commercengine/checkout");
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
  });

  onDestroy(() => {
    if (typeof window !== "undefined") {
      import("@commercengine/checkout").then(({ destroyCheckout }) => {
        destroyCheckout();
      });
    }
  });
</script>

<slot />
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `publicStorefront()` for cart/auth/order flows | Use `serverStorefront(cookies)` on server or `clientStorefront()` in browser |
| CRITICAL | Importing `$lib/server/storefront` in a `.svelte` component | Server storefront must only be used in `+page.server.ts`, `+layout.server.ts`, `hooks.server.ts`, or `+server.ts` |
| HIGH | Missing bootstrap in root layout | Add `onMount` bootstrap in `+layout.svelte` |
| HIGH | Forgetting to pass `cookies` to `serverStorefront()` | Always pass `cookies` from the load/action/hook context — there is no global cookie access |
| HIGH | Using `ssr-utils` directly instead of the first-party wrapper | Use `@commercengine/storefront/sveltekit` and `/sveltekit/server` |
| MEDIUM | Using server load when universal load suffices for public reads | Use `+page.ts` (universal) for public catalog reads — they work on both server and client |
