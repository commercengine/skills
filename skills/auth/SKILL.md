---
name: ce-auth
description: Commerce Engine authentication and user management. Anonymous sessions, OTP login, password auth, token refresh, profile management, and account flows.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.1.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Authentication & User Management

> **Prerequisite**: SDK must be initialized. See `setup/` if not done.

## When to Implement Auth Directly

If using Hosted Checkout, login and registration are usually handled inside the checkout drawer. Build custom auth UI only when the storefront needs logged-in state outside checkout, for example:

- account pages
- saved addresses
- order history
- loyalty or wallet pages
- a persistent signed-in header state

## Current Mental Model

- Public product and category reads can use `public()` and do not require anonymous auth.
- Live cart, account, and checkout flows should use the session client.
- In managed session mode, the SDK can bootstrap anonymous auth automatically on the first token-required request.
- If you need the anonymous session eagerly, call `sdk.ensureAccessToken()` once during startup.

## Quick Reference

| Auth Method | Endpoint | Use Case |
|-------------|----------|----------|
| Anonymous bootstrap | `sdk.ensureAccessToken()` or `sdk.auth.getAnonymousToken()` | Start a live session |
| Email OTP | `sdk.auth.loginWithEmail()` -> `sdk.auth.verifyOtp()` | Passwordless email login |
| Phone OTP | `sdk.auth.loginWithPhone()` -> `sdk.auth.verifyOtp()` | Passwordless phone login |
| WhatsApp OTP | `sdk.auth.loginWithWhatsApp()` -> `sdk.auth.verifyOtp()` | Passwordless WhatsApp login |
| Password login | `sdk.auth.loginWithPassword()` | Traditional login |
| Password registration | `sdk.auth.registerWithPassword()` -> OTP verification | Password sign-up |
| Forgot password | `sdk.auth.forgotPassword()` -> `sdk.auth.resetPassword()` | Reset password with OTP |
| Token refresh | `sdk.auth.refreshToken()` | Renew expired access token |

## Decision Tree

```
User Request
    │
    ├─ Public catalog-only page
    │   └─ No auth bootstrap needed; use public()
    │
    ├─ New visitor starting a live session
    │   └─ sdk.ensureAccessToken()
    │
    ├─ "Login" / "Sign in"
    │   ├─ Passwordless → loginWithEmail/Phone/WhatsApp() → verifyOtp()
    │   └─ Password → loginWithPassword()
    │
    ├─ "Register"
    │   └─ registerWithPassword() / registerWithPhonePassword() → verifyOtp()
    │
    ├─ "Forgot password"
    │   └─ forgotPassword() → resetPassword()
    │
    ├─ "Account" / "Profile"
    │   └─ getUserDetails() / updateUserDetails()
    │
    └─ "Token expired" / 401
        └─ refreshToken() or rely on managed session refresh
```

## User States

| State | How Created | Capabilities |
|-------|-------------|-------------|
| **Public** | `public()` accessor only | Public catalog/store/helper reads |
| **Anonymous session** | `sdk.ensureAccessToken()` or `/auth/anonymous` | Cart, checkout, analytics, session continuity |
| **Logged-in session** | OTP verification, password login, or password reset completion | All anonymous capabilities plus account, orders, addresses, loyalty |

## User ID vs Customer ID

For most stores, `user_id` and `customer_id` are the same value — one user = one customer. In B2B storefronts, one customer can have multiple users (e.g., a company account with multiple employees).

The SDK exposes helpers to fetch both:

```typescript
const userId = await sdk.getUserId();
const customerId = await sdk.getCustomerId();
```

Most SDK methods that require a user or customer ID have **parameterless overloads** that auto-resolve from the current session. For example, `sdk.cart.getUserCart()` (no params) fetches the cart for the logged-in user automatically. Only pass IDs explicitly when operating on behalf of a different user (admin scenarios).

## Key Patterns

### Start a live anonymous session

```typescript
const sdk = storefront.session();
await sdk.ensureAccessToken();
```

Call this once during startup if you want eager bootstrap. Do not repeat it before ordinary session-aware cart, order, or customer calls.

### OTP Login Flow (Email)

```typescript
const { data, error } = await sdk.auth.loginWithEmail({
  email: "user@example.com",
  register_if_not_exists: true,
});

if (error) throw error;

const { otp_token, otp_action } = data!;

const { error: verifyError } = await sdk.auth.verifyOtp({
  otp: "123456",
  otp_token,
  otp_action,
});

if (verifyError) throw verifyError;
```

### Password Login

```typescript
const { data, error } = await sdk.auth.loginWithPassword({
  email: "user@example.com",
  password: "securepassword",
});

if (error) throw error;
```

### Password Registration

Password registration is OTP-first in the latest contract.

```typescript
const { data, error } = await sdk.auth.registerWithPassword({
  email: "user@example.com",
  password: "securepassword",
  confirm_password: "securepassword",
});

if (error) throw error;

await sdk.auth.verifyOtp({
  otp: "123456",
  otp_token: data!.otp_token,
  otp_action: data!.otp_action,
});
```

### Forgot Password

```typescript
const { data, error } = await sdk.auth.forgotPassword({
  email: "user@example.com",
});

if (error) throw error;

await sdk.auth.resetPassword({
  otp: "123456",
  otp_token: data!.otp_token,
  new_password: "newPass123",
  confirm_password: "newPass123",
});
```

### User Profile

Auth client methods require explicit IDs (no parameterless overloads). Use `sdk.getUserId()` to get the current user's ID:

```typescript
const userId = await sdk.getUserId();

const { data: userData } = await sdk.auth.getUserDetails({ id: userId! });

await sdk.auth.updateUserDetails({ id: userId! }, {
  first_name: "Jane",
  last_name: "Doe",
});
```

## Key Feature: `register_if_not_exists`

Setting `register_if_not_exists: true` on login endpoints eliminates separate login and sign-up entry points. If the user does not exist, Commerce Engine creates the account and continues the OTP flow.

This is also the recommended approach for checkout auth — there is no separate "guest checkout" flow. The user provides their email or phone, gets verified via OTP, and the account is created if needed. This path of least resistance ensures address verification and order attribution. Hosted Checkout handles this entire flow automatically.

## Logout

The SDK exposes `sdk.auth.logout()` for custom logout buttons:

```typescript
const { data, error } = await sdk.auth.logout();
```

`logout()` does **not** clear tokens — it returns new tokens with reduced privileges. The session continues with full continuity (same cart, same analytics trail), but the user is no longer in a logged-in state. The SDK's `onTokensUpdated` callback fires with the new tokens, and the two-way sync with Hosted Checkout (when using `authMode: "provided"`) propagates them automatically — no manual token clearing or `updateTokens("", "")` needed.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `public()` for auth or account operations | Use the session client |
| CRITICAL | Assuming every page needs anonymous auth | Public reads do not; only live session flows do |
| HIGH | Building separate login and sign-up flows by default | Prefer `register_if_not_exists: true` where appropriate |
| HIGH | Forgetting `otp_token` and `otp_action` between steps | Persist both and pass them to `verifyOtp()` |
| HIGH | Storing tokens insecurely or with the wrong storage for the runtime | Use managed session storage appropriate to the environment |
| MEDIUM | Ignoring `{ data, error }` | Always check `error` before using `data` |

## See Also

- `setup/` - SDK installation and session storage
- `cart-checkout/` - Checkout flows and Hosted Checkout
- `orders/` - Logged-in order flows
- `ssr-patterns/` - Server Actions/functions for auth mutations (Next.js, TanStack Start)

## Documentation

- **Authentication Guide**: https://www.commercengine.io/docs/storefront/authentication
- **My Account**: https://www.commercengine.io/docs/storefront/my-account
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/get-anonymous-token
