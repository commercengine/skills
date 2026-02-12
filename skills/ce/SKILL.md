---
name: ce
description: Commerce Engine router. Use when user asks about building a storefront, setting up the SDK, authentication, products, cart, checkout, orders, webhooks, subscriptions, or Next.js e-commerce patterns. Automatically routes to the specific skill based on their task.
---

# Commerce Engine Skills Router

Based on what you're trying to do, here's the right skill to use:

## By Task

**Setting up the SDK** → Use `ce-setup`
- Framework detection and SDK install
- Token storage selection
- Environment variables (`CE_STORE_ID`, `CE_API_KEY`)

**Authentication & login** → Use `ce-auth`
- Anonymous auth, OTP login (email/phone/WhatsApp)
- Password auth, token refresh
- User profile management

**Products & catalog** → Use `ce-catalog`
- Products, variants, SKUs
- Categories, faceted search
- Reviews, recommendations

**Cart & checkout** → Use `ce-cart-checkout`
- Cart CRUD, coupons, loyalty points
- Fulfillment options, hosted checkout
- Payment gateway integration

**Orders & returns** → Use `ce-orders`
- Create orders from cart
- Shipment tracking, payment status
- Cancellation, returns flow

**Webhooks & events** → Use `ce-webhooks`
- 14 event types (order, payment, shipment, subscription)
- Signature verification
- Async processing patterns

**Subscriptions** → Use `ce-subscriptions`
- Standard & custom plans
- Billing/shipping cycles, trials
- Pause, revoke, coupon integration

**Next.js patterns** → Use `ce-nextjs-patterns`
- `storefront()` universal function
- `CookieTokenStorage` for SSR
- Server Actions, SSG, ISR

## Decision Tree

```
User Request
    │
    ├─ "Set up SDK" / "Add Commerce Engine"  → ce-setup
    ├─ "Login" / "Auth" / "OTP"              → ce-auth
    ├─ "Products" / "Categories" / "Search"  → ce-catalog
    ├─ "Cart" / "Checkout" / "Payments"      → ce-cart-checkout
    ├─ "Orders" / "Returns" / "Shipments"    → ce-orders
    ├─ "Webhooks" / "Events" / "Sync"        → ce-webhooks
    ├─ "Subscriptions" / "Recurring"         → ce-subscriptions
    └─ "Next.js" / "SSR" / "Server Actions"  → ce-nextjs-patterns
```

## Quick Navigation

If you know your task, you can directly access:
- `/ce-setup` - SDK setup & framework detection
- `/ce-auth` - Authentication & user management
- `/ce-catalog` - Products & categories
- `/ce-cart-checkout` - Cart, checkout & payments
- `/ce-orders` - Order management
- `/ce-webhooks` - Webhook events & syncing
- `/ce-subscriptions` - Subscription management
- `/ce-nextjs-patterns` - Next.js patterns

Or describe what you need and I'll recommend the right one.
