---
name: ce-auth
description: Commerce Engine authentication and user management. Anonymous auth, OTP login (email/phone/WhatsApp), password auth, token refresh, user profiles, and customer groups.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Authentication & User Management

> **Prerequisite**: SDK must be initialized. See `setup/` if not done.

## When to Implement Auth Directly

**If using Hosted Checkout (recommended):** Login/registration is handled inside the checkout drawer automatically. You do **not** need to build login UI or call the login/OTP endpoints below unless your storefront has features that require a logged-in user *outside* of placing orders — for example, a saved addresses page, order history, wishlists, or a loyalty dashboard.

Anonymous auth (`getAnonymousToken`) is still always required — it's handled by `StorefrontSDKInitializer` in Next.js or manually in other frameworks (see `setup/`).

When the user logs in via Hosted Checkout, tokens are synced back to the SDK automatically (via `onTokensUpdated`), so the rest of your app can use `getUserDetails()`, `listOrders()`, etc. without any additional auth work.

**Build your own login UI only when:**
- Your storefront has account pages (profile, addresses, orders) accessible before checkout
- You want a persistent login state in the header/nav (e.g., "Hi, Jane" with user menu)
- You need auth-gated pages that don't involve the checkout flow

## Quick Reference

| Auth Method | Endpoint | Use Case |
|-------------|----------|----------|
| Anonymous | `sdk.auth.getAnonymousToken()` | Every new visitor, required first step |
| Email OTP | `sdk.auth.loginWithEmail()` → `sdk.auth.verifyOtp()` | Passwordless email login |
| Phone OTP | `sdk.auth.loginWithPhone()` → `sdk.auth.verifyOtp()` | Passwordless phone login |
| WhatsApp OTP | `sdk.auth.loginWithWhatsApp()` → `sdk.auth.verifyOtp()` | Passwordless WhatsApp login |
| Password | `sdk.auth.loginWithPassword()` | Traditional email/password login |
| Token Refresh | `sdk.auth.refreshToken()` | Renew expired access token |

## Decision Tree

```
User Request
    │
    ├─ New visitor / first load
    │   └─ sdk.auth.getAnonymousToken()
    │
    ├─ "Login" / "Sign in"
    │   ├─ Passwordless (recommended)
    │   │   ├─ Email → loginWithEmail() → verifyOtp()
    │   │   ├─ Phone → loginWithPhone() → verifyOtp()
    │   │   └─ WhatsApp → loginWithWhatsApp() → verifyOtp()
    │   └─ Password → loginWithPassword()
    │
    ├─ "User profile" / "Account"
    │   └─ sdk.auth.getUserDetails() / sdk.auth.updateUserDetails()
    │
    └─ "Token expired" / 401 error
        └─ sdk.auth.refreshToken()
```

## User States

| State | How Created | Capabilities |
|-------|-------------|-------------|
| **Anonymous** | `POST /auth/anonymous` with API key | Browse catalog, manage cart, analytics tracking |
| **Logged-in** | OTP verification or password login | All anonymous + orders, addresses, profile, loyalty |

## Key Patterns

### Anonymous Auth (Required First Step)

```typescript
const { data, error } = await sdk.auth.getAnonymousToken();
// Tokens are automatically stored and managed
// Now the user can browse products, add to cart, etc.
```

### OTP Login Flow (Email)

```typescript
// 1. Initiate login (also registers new users automatically)
const { data, error } = await sdk.auth.loginWithEmail({
  email: "user@example.com",
  register_if_not_exists: true,  // Seamless login + registration
});

if (error) return handleError(error);

const { otp_token, otp_action } = data;

// 2. User enters OTP from their email...

// 3. Verify OTP
const { data: authData, error: verifyError } = await sdk.auth.verifyOtp({
  otp: "123456",           // From user input
  otp_token: otp_token,    // From step 1
  otp_action: otp_action,  // From step 1
});

// Tokens are automatically updated — user is now logged in
```

### OTP Login Flow (Phone)

```typescript
const { data, error } = await sdk.auth.loginWithPhone({
  phone: "9876543210",
  country_code: "+91",
  register_if_not_exists: true,
});

// Then verify with sdk.auth.verifyOtp() same as email flow
```

### Password Login

```typescript
const { data, error } = await sdk.auth.loginWithPassword({
  email: "user@example.com",
  password: "securepassword",
});

// Tokens automatically managed on success
```

### Token Refresh

```typescript
// The SDK handles this automatically with tokenStorage
// For manual refresh:
const { data, error } = await sdk.auth.refreshToken({
  refresh_token: storedRefreshToken,
});
```

### User Profile

```typescript
// Get user details
const { data, error } = await sdk.auth.getUserDetails({ id: userId });

// Update user details
const { data, error } = await sdk.auth.updateUserDetails({ id: userId }, {
  first_name: "Jane",
  last_name: "Doe",
});
```

### Password Management

```typescript
// Change password (logged-in user)
const { data, error } = await sdk.auth.changePassword({
  old_password: "currentPass",
  new_password: "newPass",
  confirm_password: "newPass",
});

// Forgot password flow
const { data } = await sdk.auth.forgotPassword({ email: "user@example.com" });
// Returns otp_token → user enters OTP → then:
const { data: resetData } = await sdk.auth.resetPassword({
  otp_token: data.otp_token,
  new_password: "newPass",
});
```

## Key Feature: `register_if_not_exists`

Setting `register_if_not_exists: true` on login endpoints eliminates the need for separate "Login" vs "Sign Up" flows. If the user doesn't exist, Commerce Engine automatically creates their account and sends an OTP for verification. Every user starts the same way.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Calling API without anonymous auth | Always call `sdk.auth.getAnonymousToken()` first |
| CRITICAL | Storing tokens insecurely | Use `CookieTokenStorage` (SSR) or `BrowserTokenStorage` (SPA) — never plain `localStorage` without XSS protection |
| HIGH | Separate login/register flows | Use `register_if_not_exists: true` for unified flow |
| HIGH | Forgetting `otp_token` from step 1 | Must pass both `otp_token` and `otp_action` from login response to `verifyOtp()` |
| HIGH | Not handling 401 errors | Implement token refresh — if refresh fails, re-authenticate |
| MEDIUM | Ignoring `{ data, error }` pattern | Always check `error` before using `data` |

## See Also

- `setup/` - SDK installation & token storage
- `cart-checkout/` - Cart requires auth token
- `orders/` - Orders require logged-in user
- `nextjs-patterns/` - Server Actions for auth mutations

## Documentation

- **Authentication Guide**: https://www.commercengine.io/docs/storefront/authentication
- **My Account**: https://www.commercengine.io/docs/storefront/my-account
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/get-anonymous-token
