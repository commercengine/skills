# Cookie-Based Token Management in Next.js

## How It Works

The `@commercengine/storefront-sdk-nextjs` package uses `CookieTokenStorage` under the hood. Tokens are stored in HTTP cookies, making them accessible in both server and client contexts.

## Token Flow

```
1. First Visit
   Browser → Root Layout (StorefrontSDKInitializer)
   → SDK calls auth.getAnonymousToken()
   → Tokens saved to cookies (access_token, refresh_token)

2. Server Component Render
   Request cookies → storefront(cookies()) → SDK reads tokens from cookies
   → Makes authenticated API calls → Returns data

3. Server Action (Login)
   User submits form → Server Action with storefront(cookies())
   → SDK calls auth.loginWithEmail() → verifyOtp()
   → New tokens written to cookies (upgrade from anonymous to logged-in)

4. Client Component
   storefront() → SDK reads tokens from browser cookies
   → Makes authenticated API calls
   → On token refresh, new tokens written to browser cookies

5. Token Refresh (Automatic)
   SDK detects expired access_token → Uses refresh_token to get new pair
   → New tokens written to cookies transparently
```

## StorefrontSDKInitializer

This component MUST be in your root layout. It:
1. Creates anonymous tokens for new visitors (writes to cookies)
2. Maintains session continuity across page navigations
3. Handles automatic token refresh on the client side

```tsx
// app/layout.tsx
import { StorefrontSDKInitializer } from "@commercengine/storefront-sdk-nextjs/client";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <StorefrontSDKInitializer />
        {children}
      </body>
    </html>
  );
}
```

## Build-Time Token Caching

For SSG with `generateStaticParams()`, the SDK needs tokens even at build time. Enable caching to avoid re-authenticating on every page:

```env
NEXT_BUILD_CACHE_TOKENS=true
```

This caches the anonymous token during the build process, dramatically speeding up static generation of product pages.

## Cookie Configuration

The Next.js SDK sets cookies with sensible defaults:

| Property | Default | Notes |
|----------|---------|-------|
| `secure` | `true` in production | HTTPS only |
| `sameSite` | `Lax` | Prevents CSRF |
| `path` | `/` | Available across all routes |
| `httpOnly` | `false` | Must be readable by client SDK |

## Security Considerations

- **API Key** (`NEXT_PUBLIC_API_KEY`) is safe for client-side — it's a storefront-scoped key
- **Tokens** are stored in cookies, not localStorage — accessible during SSR
- **Token refresh** happens automatically — no manual intervention needed
- **Logout** clears all token cookies
