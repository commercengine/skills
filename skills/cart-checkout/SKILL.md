---
name: ce-cart-checkout
description: Commerce Engine cart management, checkout flow, and payment integration. Cart CRUD, coupons, loyalty points, fulfillment, hosted checkout, and payment gateways.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Cart, Checkout & Payments

> **Prerequisite**: SDK initialized and anonymous auth completed. See `setup/`.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| Create cart | `sdk.cart.createCart({ ... })` |
| Get cart | `sdk.cart.retrieveCart(cartId)` |
| Get cart by user | `sdk.cart.retrieveCartByUserId(userId)` |
| Add item | `sdk.cart.addCartItem(cartId, { product_id, variant_id, quantity })` |
| Update item | `sdk.cart.updateCartItem(cartId, { product_id, variant_id, quantity })` |
| Remove item | `sdk.cart.removeCartItem(cartId, { product_id, variant_id })` |
| Apply coupon | `sdk.cart.applyCoupon(cartId, { coupon_code })` |
| Remove coupon | `sdk.cart.removeCoupon(cartId, { coupon_code })` |
| List coupons | `sdk.cart.retrieveAllCoupons()` |
| Update address | `sdk.cart.updateCartAddress(cartId, { ... })` |
| Check fulfillment | `sdk.cart.checkFulfillment(cartId)` |
| Get fulfillment options | `sdk.cart.retrieveFulfillmentOptions(cartId)` |
| Create order | `sdk.order.createOrder({ ... })` |

## Decision Tree

```
User Request
    │
    ├─ "Add to cart"
    │   ├─ Cart exists? → sdk.cart.addCartItem()
    │   └─ No cart? → sdk.cart.createCart() → then addCartItem()
    │
    ├─ "View cart"
    │   └─ sdk.cart.retrieveCart(cartId) or retrieveCartByUserId(userId)
    │
    ├─ "Apply coupon" / "Discount"
    │   ├─ List available → sdk.cart.retrieveAllCoupons()
    │   └─ Apply → sdk.cart.applyCoupon(cartId, { coupon_code })
    │
    ├─ "Checkout"
    │   ├─ Custom checkout → See "Full Checkout Flow" below
    │   └─ Hosted checkout → See references/hosted-checkout.md
    │
    └─ "Payment"
        └─ sdk.order.createOrder() → payment_info → gateway redirect
```

## Cart Structure

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

## Key Patterns

### Create Cart and Add Items

```typescript
// Create a cart
const { data: cart, error } = await sdk.cart.createCart({
  items: [
    { product_id: "prod_123", variant_id: "var_456", quantity: 2 },
  ],
});

// Add more items to existing cart
const { data, error } = await sdk.cart.addCartItem(cart.id, {
  product_id: "prod_789",
  variant_id: "var_012",
  quantity: 1,
});
```

### Update and Remove Items

```typescript
// Update quantity
const { data, error } = await sdk.cart.updateCartItem(cartId, {
  product_id: "prod_123",
  variant_id: "var_456",
  quantity: 3,
});

// Remove item
const { data, error } = await sdk.cart.removeCartItem(cartId, {
  product_id: "prod_123",
  variant_id: "var_456",
});
```

### Apply Coupon

```typescript
// List available coupons
const { data: coupons } = await sdk.cart.retrieveAllCoupons();

// Apply a coupon
const { data, error } = await sdk.cart.applyCoupon(cartId, {
  coupon_code: "SAVE20",
});

// Remove coupon
const { data, error } = await sdk.cart.removeCoupon(cartId, {
  coupon_code: "SAVE20",
});
```

### Full Checkout Flow

See `references/checkout-flow.md` for the complete step-by-step flow. Summary:

1. **Review cart** → `sdk.cart.retrieveCart(cartId)`
2. **Set addresses** → `sdk.cart.updateCartAddress(cartId, { billing_address, shipping_address })`
3. **Check fulfillment** → `sdk.cart.checkFulfillment(cartId)`
4. **Get fulfillment options** → `sdk.cart.retrieveFulfillmentOptions(cartId)`
5. **Apply discounts** → coupons, loyalty points, credit balance
6. **Create order** → `sdk.order.createOrder({ cart_id, payment_method })`
7. **Process payment** → Use `payment_info` from order response
8. **Poll payment status** → `sdk.order.retrievePaymentStatus(orderNumber)`

### Hosted Checkout (Quick Integration)

For a pre-built checkout experience, use the hosted checkout SDK. See `references/hosted-checkout.md`.

```typescript
// React
import { initCheckout } from "@commercengine/checkout/react";

initCheckout({
  storeId: "store_xxx",
  apiKey: "ak_xxx",
  environment: "production",
});
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Cart expired | Check `expires_at` — create new cart if expired |
| HIGH | Adding product instead of variant | When product `has_variant: true`, must specify `variant_id` |
| HIGH | Coupon requires login | Some coupons require logged-in user — prompt login first |
| HIGH | Missing address before checkout | Must set billing/shipping address before creating order |
| MEDIUM | Not checking fulfillment | Always check `checkFulfillment()` after setting address |
| MEDIUM | Ignoring `to_be_paid` | Display `to_be_paid` not `grand_total` — it accounts for loyalty/credit |

## See Also

- `setup/` - SDK initialization
- `auth/` - Login required for some cart operations
- `catalog/` - Product data for cart items
- `orders/` - After checkout, order management
- `subscriptions/` - Recurring orders bypass standard cart flow

## Documentation

- **Cart Guide**: https://www.commercengine.io/docs/storefront/cart
- **Checkout Flow**: https://www.commercengine.io/docs/storefront/checkout
- **Hosted Checkout**: https://www.commercengine.io/docs/hosted-checkout/overview
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/create-cart
