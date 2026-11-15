---
name: ce-auth
description: Commerce Engine authentication and user management. Anonymous sessions, OTP login, password auth, token refresh, forgot password and forgot username (account recovery), profile management, customer attributes, and account flows.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.2.0"
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
| Forgot username | `sdk.auth.forgotUsername()` -> `sdk.auth.verifyForgotUsernameOtp()` | Recover an account by its KYC document numbers; verifying signs in |
| Customer attributes | `sdk.store.listCustomerAttributes()`, `sdk.auth.listUserAttributes()` / `updateUserAttributes()` | Store-defined profile fields and the customer's values |
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
    │   ├─ Password → registerWithPassword() (email or phone + password) → verifyOtp()
    │   └─ Passwordless → registerWithEmail/Phone/Whatsapp() → verifyOtp()
    │
    ├─ "Forgot password"
    │   └─ forgotPassword() → resetPassword()
    │
    ├─ "Forgot username" / "Can't remember my login" (B2B, KYC)
    │   └─ forgotUsername() → verifyForgotUsernameOtp() (again with user_id if it returns users)
    │
    ├─ "Account" / "Profile"
    │   ├─ getUserDetails() / updateUserDetails()
    │   └─ Custom profile fields → store.listCustomerAttributes() → listUserAttributes() / updateUserAttributes()
    │
    └─ "Token expired" / 401
        └─ refreshToken() or rely on managed session refresh
```

## User States

| State | How Created | Capabilities |
|-------|-------------|-------------|
| **Public** | `public()` accessor only | Public catalog/store/helper reads |
| **Anonymous session** | `sdk.ensureAccessToken()` or `/auth/anonymous` | Cart, checkout, analytics, session continuity |
| **Logged-in session** | OTP verification, password login, password reset completion, or forgot-username verification | All anonymous capabilities plus account, orders, addresses, loyalty |

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

`resetPassword()` returns the `user` with new tokens (managed sessions store them), so a completed reset leaves the shopper signed in.

### Forgot Username (Account Recovery by KYC Documents)

For stores with KYC, a shopper who doesn't remember which email or phone the account signs in with can recover it with document numbers: the code goes to the account's registered contact, and verifying it signs them in. The `linked` customer attributes are the documents a lookup can use:

```typescript
const { data: linked } = await sdk.store.listCustomerAttributes({ kind: "linked" });
// linked.attributes → [{ id, slug, name, attribute_type, kind: "linked", ... }]
// Render one input per linked attribute (e.g. a GSTIN), then send the ones the shopper filled.

const { data, error } = await sdk.auth.forgotUsername({
  // 1 to 5 filters: a linked attribute's id and the shopper's document number for it
  filters: [{ attribute_id: gstinAttributeId, value: "27AAPFU0939F1ZV" }],
});

if (error) throw error;
// Same response whether or not an account matched: never tell the shopper which.
// When exactly one customer matches, the code goes to its sign-in phone (or email).

const { data: verified, error: verifyError } = await sdk.auth.verifyForgotUsernameOtp({
  otp: "123456",
  otp_token: data!.otp_token,
});

if (verifyError) throw verifyError;

if ("users" in verified!) {
  // Business account with several active users: not signed in yet.
  // users → [{ id, first_name, last_name, email (masked), phone (masked) }], primary first
  const chosen = verified.users[0]!;
  await sdk.auth.verifyForgotUsernameOtp({
    otp: "123456",
    otp_token: data!.otp_token,
    user_id: chosen.id,
  });
} else {
  // Signed in, as with verifyOtp(): verified.user, access_token, refresh_token
}
```

`verifyForgotUsernameOtp()` signs in like `verifyOtp()`: managed sessions store the returned tokens, and the visitor's cart and attribute values move to the account. The `users` response carries no tokens and leaves the session as it is; the code stays valid until a call with `user_id` signs one in. Both calls need the same session (access token) the code was requested with. A wrong, expired or used-up code and a lookup that matched no account all get the same error: request a new code.

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

On `User`, `business`, `customer_group` and `customer_id` are optional: guard them (`user.customer_group?.id`), or use `sdk.getCustomerId()` for the customer ID. The customer's own key-value data is `user.metadata`, shared by all its users (`Business` no longer has `metadata`). In `updateUserDetails()`, each `metadata` key sent is set, a key sent as `null` is removed, and keys not sent are kept (at most 50 keys and 16 KB).

### Customer Attributes

Customer attributes are store-defined profile fields. `sdk.store.listCustomerAttributes()` lists them (on the public and the session SDK; the API key alone is enough). Shoppers can set the `custom` ones that aren't `is_readonly`; `linked` ones are KYC documents (see Forgot Username) and `system` ones are read-only.

```typescript
// Fields for a profile form
const { data: editable } = await sdk.store.listCustomerAttributes({
  kind: "custom",
  is_readonly: false,
});
// editable.attributes → [{ id, slug, name, attribute_type: "String" | "Integer" | "Number" | "Boolean" | "List" | "Object" | "DateTime", ... }]

const userId = await sdk.getUserId();

// The customer's values: every attribute, value null where none is set
const { data: values } = await sdk.auth.listUserAttributes({ id: userId! });

// Set values by slug or attribute_id; null, "" or only spaces removes a value
const { data, error } = await sdk.auth.updateUserAttributes({ id: userId! }, [
  { slug: "preferred_language", value: "en" },
  { attribute_id: "01H9ATTR12345ABCDE", value: true },
  { slug: "nickname", value: null },
]);
```

All items in `updateUserAttributes()` are applied or none: a 400 lists each refused item's errors under `errors.attributes`, keyed by its position. `listUserAttributes()` and `updateUserAttributes()` also work with an anonymous token, using the visitor's user ID: the values move to the customer when the visitor signs in or signs up. The `id` must be the token's own user; any other ID gets a 404. In a business account every user of the customer shares the same values. `updateUserDetails()` also accepts `attributes` (the same items) to set them in the same change as the profile.

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
| HIGH | Assuming `verifyForgotUsernameOtp()` always signs in | Check `"users" in data`; for a business account with several users, call it again with the chosen `user_id` |
| HIGH | Telling the shopper whether a forgot-username lookup matched | `forgotUsername()` answers the same either way; show one neutral "if an account matches, we sent a code" message |
| MEDIUM | Reading `user.business.metadata`, or `user.customer_id` / `customer_group` / `business` unguarded | Use `user.metadata`; guard the optional fields or use `sdk.getCustomerId()` |
| MEDIUM | Ignoring `{ data, error }` | Always check `error` before using `data` |

## See Also

- `setup/` - SDK installation and session storage
- `cart-checkout/` - Checkout flows and Hosted Checkout
- `orders/` - Logged-in order flows
- `ssr-patterns/` - Server Actions/functions for auth mutations (Next.js, TanStack Start)
- `analytics/` - Identifying the shopper in browser analytics after login

## Documentation

- **Authentication Guide**: https://www.commercengine.io/docs/storefront/authentication
- **My Account**: https://www.commercengine.io/docs/storefront/my-account
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/get-anonymous-token
- **Account Recovery**: https://llm-docs.commercengine.io/storefront/operations/forgot-username, https://llm-docs.commercengine.io/storefront/operations/verify-forgot-username-otp
- **Customer Attributes**: https://llm-docs.commercengine.io/storefront/operations/list-attributes, https://llm-docs.commercengine.io/storefront/operations/update-user-attributes
