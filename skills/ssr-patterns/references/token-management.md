# Token Management in SSR Frameworks

## How It Works

The `@commercengine/storefront` package handles token storage automatically for Next.js and TanStack Start via the `createSSRFactory` abstraction from `@commercengine/ssr-utils`.

**Three-layer model:**

- `publicStorefront()` — never touches session lifecycle. API-key-backed only.
- `clientStorefront()` — browser-side session client with cookie-backed token storage.
- `serverStorefront()` — server-side session client that reads/writes request cookies.

The client and server storages share the same cookie format so hydration and navigation preserve the same session.

## Token Flow

```
1. First visit
   Browser → Root Layout renders
   → StorefrontBootstrap calls storefront.bootstrap()
   → bootstrap() establishes the client session once
   → Anonymous session tokens are created and written to cookies
   → Subsequent server reads see the session via request cookies

2. Public server rendering / pre-rendering
   publicStorefront()
   → API-key-backed reads only
   → No token reads or writes
   → Safe for build time, ISR, SSG

3. Server-side session read (Next.js)
   await storefront.serverStorefront()
   → Dynamically imports next/headers, reads cookies
   → SDK reads tokens from request cookies
   → May refresh or persist updated tokens
   → Deduped per request via React cache()

4. Server-side session read (TanStack Start)
   serverStorefront() — from @commercengine/storefront/tanstack-start/server
   → Reads cookies via getCookie() from @tanstack/react-start/server
   → Fresh per call (each server function invocation)

5. Client-side session
   storefront.clientStorefront()
   → SDK reads matching session cookies in the browser
   → May refresh tokens and keep the browser session current

6. Login / logout
   Server function or client session flow updates tokens
   → Cookies update (server-side via setCookie, client-side via browser cookies)
   → Returning renders continue the same session
```

## Returning Users & Cookie Hydration

When a user returns to the site, their session cookies (access + refresh tokens) are already present from the previous visit. In this case:

- `serverStorefront()` **automatically reads existing cookies** and participates in the returning user's session — whether anonymous or logged-in. No bootstrap needed for the server to see the session.
- `clientStorefront()` reads the same cookies on the browser side after hydration.
- `bootstrap()` is a **no-op** when cookies already exist. It only creates a new anonymous session on first visit.

This means Server Components can render session-aware data (account, orders, cart) for returning users on the very first render, because the cookies are sent with the request.

## Bootstrap Pattern

**Why bootstrap is required:** On a **first visit** (no cookies), the server cannot reliably establish a new anonymous session because the cookie write from a Server Component/function may not reach the browser before subsequent requests. The browser must be the one to create the initial session.

```tsx
// Pattern works for both Next.js and TanStack Start
"use client"; // Only needed in Next.js

import { useEffect } from "react";
import { storefront } from "@/lib/storefront";

export function StorefrontBootstrap() {
  useEffect(() => {
    storefront.bootstrap().catch(console.error);
  }, []);
  return null;
}
```

`bootstrap()` is deduped — concurrent calls resolve to a single operation. If a session already exists in cookies, it's a no-op.

After that eager bootstrap is in place, ordinary session-aware SDK calls such as `sdk.cart.getWishlist()`, `sdk.cart.addToWishlist()`, `sdk.cart.getUserCart()`, and `sdk.customer.listAddresses()` do not need their own manual `ensureAccessToken()` call.

## Configuration

Both `createNextjsStorefront()` and `createTanStackStartStorefront()` accept `tokenStorageOptions`:

| Property | Default | Notes |
|----------|---------|-------|
| `prefix` | `"ce_"` | Cookie name prefix |
| `maxAge` | `2592000` (30 days) | Cookie max-age in seconds |
| `path` | `"/"` | Available across all routes |
| `domain` | — | Cookie domain (defaults to request domain) |
| `secure` | auto-detected | `true` in production (HTTPS) |
| `sameSite` | `"lax"` | CSRF protection |

The client and server cookie configurations are automatically aligned — you don't need to configure them separately.

## Security Notes

- `NEXT_PUBLIC_API_KEY` / `VITE_API_KEY` are safe for client-side storefront use — they are scoped to public storefront operations.
- Public reads should use `publicStorefront()` instead of creating anonymous sessions unnecessarily.
- Token refresh is automatic in managed session mode.
- Logout should clear session tokens so checkout or account UI does not hold stale state.
- Cookies are not `httpOnly` — the browser session client must be able to read/write them.
