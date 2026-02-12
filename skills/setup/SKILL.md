---
name: ce-setup
description: Set up the Commerce Engine TypeScript SDK in any project. Framework detection, token storage selection, environment variables, and migration guidance.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Setting Up Commerce Engine

This skill sets up the Commerce Engine TypeScript SDK for your project.

## Quick Reference

| Step | Action |
|------|--------|
| 1. Detect framework | Check `package.json` and config files |
| 2. Install SDK | `npm install @commercengine/storefront-sdk` |
| 3. Choose token storage | `BrowserTokenStorage` (SPA), `CookieTokenStorage` (SSR), `MemoryTokenStorage` (Node.js) |
| 4. Set env vars | `CE_STORE_ID` and `CE_API_KEY` |
| 5. Initialize SDK | Create `lib/storefront.ts` |
| 6. Get anonymous token | `sdk.auth.getAnonymousToken()` before any API call |

## Framework Detection

Check `package.json` and config files to identify the framework:

| Indicator | Framework | SDK Package | Token Storage |
|-----------|-----------|-------------|---------------|
| `next` in deps + `next.config.*` | Next.js | `@commercengine/storefront-sdk-nextjs` | `CookieTokenStorage` |
| `vite.config.ts` + `@vitejs/plugin-react` | React SPA | `@commercengine/storefront-sdk` | `BrowserTokenStorage` |
| `vite.config.ts` + `@vitejs/plugin-vue` | Vue SPA | `@commercengine/storefront-sdk` | `BrowserTokenStorage` |
| `svelte.config.js` | Svelte/SvelteKit | `@commercengine/storefront-sdk` | `BrowserTokenStorage` |
| `solid-js` in deps | Solid | `@commercengine/storefront-sdk` | `BrowserTokenStorage` |
| `express` in deps | Express/Node.js | `@commercengine/storefront-sdk` | `MemoryTokenStorage` |
| None of above | Vanilla JS | `@commercengine/storefront-sdk` | `BrowserTokenStorage` |

## Decision Tree

```
User Request: "Set up Commerce Engine" / "Add e-commerce"
    │
    ├─ Read package.json + config files
    │
    ├─ Next.js detected?
    │   ├─ YES → Install @commercengine/storefront-sdk-nextjs
    │   │        → Use storefront() function + StorefrontSDKInitializer
    │   │        → See ce-nextjs-patterns for advanced usage
    │   └─ NO → Install @commercengine/storefront-sdk
    │
    ├─ Choose token storage based on framework
    │
    ├─ Set environment variables
    │
    └─ Initialize SDK + get anonymous token
```

## Setup by Framework

### Next.js (App Router)

```bash
npm install @commercengine/storefront-sdk-nextjs
```

```typescript
// lib/storefront.ts
export { storefront } from "@commercengine/storefront-sdk-nextjs";
```

```typescript
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

```env
# .env.local
NEXT_PUBLIC_STORE_ID=your-store-id
NEXT_PUBLIC_API_KEY=your-api-key
```

### React / Vue / Svelte / Solid (SPA)

```bash
npm install @commercengine/storefront-sdk
```

```typescript
// lib/storefront.ts
import { StorefrontSDK, Environment, BrowserTokenStorage } from "@commercengine/storefront-sdk";

export const sdk = new StorefrontSDK({
  storeId: import.meta.env.VITE_STORE_ID,
  environment: import.meta.env.PROD ? Environment.Production : Environment.Staging,
  apiKey: import.meta.env.VITE_API_KEY,
  tokenStorage: new BrowserTokenStorage("myapp_"),
});
```

```env
# .env
VITE_STORE_ID=your-store-id
VITE_API_KEY=your-api-key
```

### Node.js / Express

```bash
npm install @commercengine/storefront-sdk
```

```typescript
// src/lib/storefront.ts
import { StorefrontSDK, Environment, MemoryTokenStorage } from "@commercengine/storefront-sdk";

const sdk = new StorefrontSDK({
  storeId: process.env.CE_STORE_ID!,
  environment: process.env.NODE_ENV === "production" ? Environment.Production : Environment.Staging,
  apiKey: process.env.CE_API_KEY!,
  tokenStorage: new MemoryTokenStorage(),
  timeout: 30000,
});

export default sdk;
```

## Token Storage Guide

| Storage | Use Case | Persistence | SSR Safe |
|---------|----------|-------------|----------|
| `BrowserTokenStorage` | SPAs (React, Vue, Svelte, Solid) | localStorage | No |
| `CookieTokenStorage` | SSR frameworks (Next.js) | Cookies | Yes |
| `MemoryTokenStorage` | Server-side (Node.js, Express) | In-memory | Yes |

## First API Call

After setup, always authenticate anonymously before making any API calls:

```typescript
// Every visitor starts as anonymous
const { data, error } = await sdk.auth.getAnonymousToken();

if (error) {
  console.error("Auth failed:", error.message);
} else {
  // Tokens are automatically managed by the SDK
  // Now you can call any API
  const { data: products } = await sdk.catalog.listProducts();
}
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CE_STORE_ID` / `NEXT_PUBLIC_STORE_ID` | Yes | Your store identifier |
| `CE_API_KEY` / `NEXT_PUBLIC_API_KEY` | Yes | Storefront API key (safe for client-side) |
| `NEXT_BUILD_CACHE_TOKENS` | No | Set `true` for faster Next.js builds with token caching |

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Missing anonymous auth | Must call `sdk.auth.getAnonymousToken()` before any API call |
| CRITICAL | Exposing admin API keys | Only storefront API keys (`CE_API_KEY`) are safe for client-side |
| HIGH | Wrong token storage for SSR | Use `CookieTokenStorage` for Next.js, not `BrowserTokenStorage` |
| HIGH | Missing env vars | SDK throws if `storeId` or `apiKey` is missing — check env var names |
| MEDIUM | Missing `StorefrontSDKInitializer` | Required in Next.js root layout for automatic anonymous auth |
| MEDIUM | Wrong environment | Use `Environment.Staging` for dev, `Environment.Production` for prod |

## See Also

- `auth/` - Authentication & login flows
- `nextjs-patterns/` - Advanced Next.js patterns with `storefront()`
- `cart-checkout/` - Cart management after setup

## Documentation

- **SDK Installation**: https://www.commercengine.io/docs/sdk/installation
- **Configuration**: https://www.commercengine.io/docs/sdk/configuration
- **Token Management**: https://www.commercengine.io/docs/sdk/token-management
- **LLM Reference**: https://llm-docs.commercengine.io/sdk/
