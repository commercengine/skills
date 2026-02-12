---
name: ce-auth
description: Commerce Engine authentication and user management. Anonymous auth, OTP login (email/phone/WhatsApp), password auth, token refresh, user profiles, and customer groups.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Authentication & User Management

> **Prerequisite**: SDK must be initialized. See `setup/` if not done.

## Quick Reference

| Auth Method | Endpoint | Use Case |
|-------------|----------|----------|
| Anonymous | `sdk.auth.getAnonymousToken()` | Every new visitor, required first step |
| Email OTP | `sdk.auth.loginWithEmail()` → `sdk.auth.verifyOtp()` | Passwordless email login |
| Phone OTP | `sdk.auth.loginWithPhone()` → `sdk.auth.verifyOtp()` | Passwordless phone login |
| WhatsApp OTP | `sdk.auth.loginWithWhatsapp()` → `sdk.auth.verifyOtp()` | Passwordless WhatsApp login |
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
    │   │   └─ WhatsApp → loginWithWhatsapp() → verifyOtp()
    │   └─ Password → loginWithPassword()
    │
    ├─ "User profile" / "Account"
    │   └─ sdk.auth.getUser() / sdk.auth.updateUser()
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

const { otp_token, otp_action } = data.content;

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
const { data, error } = await sdk.auth.getUser(userId);

// Update user details
const { data, error } = await sdk.auth.updateUser(userId, {
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
  otp_token: data.content.otp_token,
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
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/anonymous-user
