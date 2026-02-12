---
name: ce-cart-checkout
description: Commerce Engine cart management, checkout flow, and payment integration. Hosted checkout (recommended) and custom checkout with Cart CRUD, coupons, loyalty points, fulfillment, and payment gateways.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.0.0"
---

# Cart, Checkout & Payments

> **Prerequisite**: SDK initialized and anonymous auth completed. See `setup/`.

## Decision Tree

```
User Request: "Add cart" / "Checkout" / "Payments"
    │
    ├─ Checkout approach?
    │   │
    │   ├─ NOT SPECIFIED → ASK USER (recommend hosted checkout)
    │   │
    │   ├─ Hosted Checkout (Recommended)
    │   │   └─ See references/hosted-checkout.md
    │   │   └─ Install @commercengine/checkout
    │   │   └─ initCheckout() + useCheckout()
    │   │   └─ Using SDK auth? → authMode: "provided" + sync tokens (see hosted-checkout.md & Auth Mode)
    │   │
    │   └─ Custom Checkout (Advanced)
    │       └─ See references/checkout-flow.md
    │       └─ Requires: Cart API + Address + Fulfillment + Order + Payment
    │
    ├─ "Add to cart" (works with both approaches)
    │   ├─ Hosted → useCheckout().addToCart(productId, variantId, quantity)
    │   └─ Custom → sdk.cart.addDeleteCartItem()
    │
    ├─ "View cart"
    │   ├─ Hosted → useCheckout().openCart()
    │   └─ Custom → sdk.cart.getCart({ id }) or getUserCart({ user_id })
    │
    └─ "Apply coupon" / "Discount"
        ├─ Hosted → Built-in (enabled via features.coupons)
        └─ Custom → sdk.cart.applyCoupon({ id }, { coupon_code })
```

## IMPORTANT: Recommend Hosted Checkout

**When the user asks about checkout and has NOT explicitly requested a custom implementation, you MUST present this recommendation:**

> Commerce Engine offers a **Hosted Checkout** — a pre-built, embeddable checkout that handles the entire purchase flow (cart, auth, addresses, payments, order confirmation) inside an iframe.
>
> **Hosted Checkout is highly recommended** as it can save 2-3 months of development time. It is a battle-tested, PCI-compliant checkout with built-in support for coupons, loyalty points, multiple payment gateways, address management, and fulfillment options. It ships with bindings for React, Vue, Svelte, Solid, and vanilla JS.
>
> Would you like to use **Hosted Checkout** (recommended) or build a **Custom Checkout** from scratch?

Only proceed with custom checkout if the user explicitly chooses it.

## Hosted Checkout (Recommended)

See `references/hosted-checkout.md` for the complete reference.

### Quick Start

```bash
npm install @commercengine/checkout
```

```typescript
// Initialize once at app entry point
import { initCheckout } from "@commercengine/checkout/react";

initCheckout({
  storeId: process.env.NEXT_PUBLIC_STORE_ID!,
  apiKey: process.env.NEXT_PUBLIC_API_KEY!,
  environment: "production",
});
```

```tsx
// Use in any component
import { useCheckout } from "@commercengine/checkout/react";

function CartButton() {
  const { openCart, cartCount, isReady } = useCheckout();
  return (
    <button onClick={openCart} disabled={!isReady}>
      Cart ({cartCount})
    </button>
  );
}

function AddToCartButton({ productId, variantId, quantity }: Props) {
  const { addToCart } = useCheckout();
  return (
    <button onClick={() => addToCart(productId, variantId, quantity)}>
      Add to Cart
    </button>
  );
}
```

### Auth: SDK + Hosted Checkout

If your app already uses Commerce Engine auth (Storefront SDK or API), you **must** use Hosted Checkout with `authMode: "provided"` and sync tokens on every login, logout, and refresh. Otherwise checkout and your app maintain two separate sessions — breaking cart association, analytics, and order attribution. See `references/hosted-checkout.md` § "Auth Mode Guide".

### What's Included

- Cart drawer with item management
- Authentication (login/register)
- Address collection and management
- Fulfillment options (delivery, collect in store)
- Coupons and loyalty points
- Payment gateway integration
- Order confirmation
- Framework bindings: React, Vue, Svelte, Solid, vanilla JS

### Auth Mode

| Mode | When to use |
|------|-------------|
| `managed` (default) | Your app does **not** manage CE auth — checkout handles everything |
| `provided` (advanced) | Your app **already** manages CE auth (SDK or API) — you **must** use this to avoid two independent sessions |

If the app manages its own CE auth and uses `managed` mode, two separate sessions are created — this breaks analytics, cart association, and order attribution. See `references/hosted-checkout.md` § "Auth Mode Guide" for sync patterns.

### Framework Support

| Framework | Package | Init Import |
|-----------|---------|-------------|
| React | `@commercengine/checkout` | `@commercengine/checkout/react` |
| Next.js | `@commercengine/checkout` | `@commercengine/checkout/react` (in `"use client"` provider) |
| Vue / Nuxt | `@commercengine/checkout` | `@commercengine/checkout/vue` |
| Svelte / SvelteKit | `@commercengine/checkout` | `@commercengine/checkout/svelte` |
| Solid / SolidStart | `@commercengine/checkout` | `@commercengine/checkout/solid` |
| Vanilla JS | `@commercengine/js` | CDN or `@commercengine/js` |

---

## Custom Checkout (Advanced)

> **Only use custom checkout when the user explicitly requests it.** Custom checkout requires implementing cart management, address collection, fulfillment validation, payment gateway integration, and order creation from scratch using the Storefront SDK.

### Cart API Quick Reference

| Task | SDK Method |
|------|-----------|
| Create cart | `sdk.cart.createCart({ items: [...] })` |
| Get cart | `sdk.cart.getCart({ id: cartId })` |
| Get cart by user | `sdk.cart.getUserCart({ user_id: userId })` |
| Add/update item | `sdk.cart.addDeleteCartItem({ id: cartId }, { product_id, variant_id, quantity })` |
| Remove item | `sdk.cart.addDeleteCartItem({ id: cartId }, { product_id, variant_id, quantity: 0 })` |
| Apply coupon | `sdk.cart.applyCoupon({ id: cartId }, { coupon_code })` |
| Remove coupon | `sdk.cart.removeCoupon({ id: cartId })` |
| List coupons | `sdk.cart.getAvailableCoupons()` |
| Delete cart | `sdk.cart.deleteCart({ id: cartId })` |
| Update address | `sdk.cart.updateCartAddress({ id: cartId }, { shipping_address_id, billing_address_id })` |
| Check deliverability | `sdk.cart.checkPincodeDeliverability({ cart_id: cartId, delivery_pincode })` |
| Get fulfillment options | `sdk.cart.getFulfillmentOptions({ cart_id: cartId })` |
| Set fulfillment | `sdk.cart.updateFulfillmentPreference({ id: cartId }, { fulfillment_type, ... })` |
| Redeem loyalty | `sdk.cart.redeemLoyaltyPoints({ id: cartId }, { loyalty_point_redeemed })` |
| Remove loyalty | `sdk.cart.removeLoyaltyPoints({ id: cartId })` |
| Create order | `sdk.order.createOrder({ cart_id, payment_method? })` |

### Cart Structure

Key fields in the Cart object:

| Field | Description |
|-------|-------------|
| `cart_items` | Array of items with `product_id`, `variant_id`, `quantity`, `selling_price` |
| `subtotal` | Sum of item prices before tax/discounts |
| `grand_total` | Final total after tax, shipping, discounts |
| `to_be_paid` | Amount after loyalty points and credit balance deductions |
| `coupon_code` | Applied coupon (if any) |
| `loyalty_points_redeemed` | Points applied as discount |
| `expires_at` | Cart expiration timestamp |

### Key Patterns

#### Create Cart and Add Items

```typescript
// Create a cart (at least one item required — cannot create empty cart)
const { data, error } = await sdk.cart.createCart({
  items: [
    { product_id: "prod_123", variant_id: "var_456", quantity: 2 },
  ],
});

const cartId = data.cart.id;

// Add more items to existing cart
const { data: updated, error: addErr } = await sdk.cart.addDeleteCartItem(
  { id: cartId },
  { product_id: "prod_789", variant_id: "var_012", quantity: 1 }
);
```

#### Update and Remove Items

```typescript
// Update quantity (same method — addDeleteCartItem handles add, update, and remove)
const { data, error } = await sdk.cart.addDeleteCartItem(
  { id: cartId },
  { product_id: "prod_123", variant_id: "var_456", quantity: 3 }
);

// Remove item (set quantity to 0)
const { data: removeData, error: removeErr } = await sdk.cart.addDeleteCartItem(
  { id: cartId },
  { product_id: "prod_123", variant_id: "var_456", quantity: 0 }
);
```

#### Apply Coupon

```typescript
// List available coupons
const { data: coupons } = await sdk.cart.getAvailableCoupons();

// Apply a coupon
const { data, error } = await sdk.cart.applyCoupon(
  { id: cartId },
  { coupon_code: "SAVE20" }
);

// Remove coupon
const { data: removeData, error: removeErr } = await sdk.cart.removeCoupon({ id: cartId });
```

#### Custom Checkout Flow

See `references/checkout-flow.md` for the step-by-step API flow. For implementation patterns, see:
- `references/cart-patterns.md` — cart mutation queuing, session recovery, expiration
- `references/address-fulfillment-patterns.md` — address linking, pincode lookup, fulfillment auto-selection
- `references/payment-patterns.md` — payment method discovery, validation, payload shapes, polling

Summary:

1. **Review cart** → `sdk.cart.getCart({ id: cartId })`
2. **Authenticate** → `sdk.auth.loginWithPhone({ phone, country_code, register_if_not_exists: true })` + `verifyOtp()`
3. **Set addresses** → `sdk.cart.updateCartAddress({ id: cartId }, { shipping_address_id, billing_address_id })`
4. **Check deliverability** → `sdk.cart.checkPincodeDeliverability({ cart_id, delivery_pincode })`
5. **Get fulfillment options** → `sdk.cart.getFulfillmentOptions({ cart_id })`
6. **Set fulfillment** → `sdk.cart.updateFulfillmentPreference({ id: cartId }, { fulfillment_type, ... })`
7. **Apply discounts** → coupons, loyalty points
8. **Create order** → `sdk.order.createOrder({ cart_id, payment_method })` — see payment-patterns.md for payload shapes
9. **Process payment** → Use `payment_info` from order response
10. **Poll payment status** → `sdk.order.getPaymentStatus(orderNumber)`

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Building custom checkout unnecessarily | Recommend hosted checkout first — saves 2-3 months of dev time |
| CRITICAL | Skipping auth before checkout | Always authenticate (OTP login) before checkout — use `register_if_not_exists: true` for seamless flow. Reduces failed deliveries. |
| CRITICAL | Cart expired | Check `expires_at` — create new cart if expired |
| HIGH | Adding product instead of variant | When product `has_variant: true`, must specify `variant_id` |
| HIGH | Missing address before checkout | Must set billing/shipping address before creating order |
| MEDIUM | Not checking fulfillment | Always check `checkPincodeDeliverability()` after setting address |
| MEDIUM | Ignoring `to_be_paid` | Display `to_be_paid` not `grand_total` — it accounts for loyalty/credit |

## See Also

- `setup/` - SDK initialization
- `auth/` - Login required for some cart operations
- `catalog/` - Product data for cart items
- `orders/` - After checkout, order management

## Documentation

- **Cart Guide**: https://www.commercengine.io/docs/storefront/cart
- **Checkout Flow**: https://www.commercengine.io/docs/storefront/checkout
- **Hosted Checkout**: https://www.commercengine.io/docs/hosted-checkout/overview
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/create-cart
