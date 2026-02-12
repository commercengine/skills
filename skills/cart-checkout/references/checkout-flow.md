# Checkout Flow: Cart to Order

Step-by-step guide for converting a cart into a placed order.

## Recommended Flow: Cart → Auth → Checkout

The user should authenticate **before** entering checkout. Since delivery requires a phone number or email, and Commerce Engine auth is OTP-based, this is seamless — the same contact info used for login also serves as verified delivery contact. This reduces failed deliveries (RTOs) caused by incorrect contact details.

```
Add to Cart → Login/Register (OTP) → Addresses → Fulfillment → Payment → Confirmation
```

Use `register_if_not_exists: true` with `loginWithPhone` or `loginWithWhatsApp` for a seamless flow — new users are automatically registered, existing users log in. No separate registration step needed.

## Prerequisites

- A valid `cart_id` with items added
- User authenticated (anonymous → OTP login/register before checkout)

## Flow

### Step 1: Review Cart

```typescript
const { data, error } = await sdk.cart.getCart({ id: cartId });

// Display: data.cart.cart_items, subtotal, grand_total, applied discounts
// Check expires_at to ensure cart hasn't expired
```

### Step 2: Authenticate (Before Checkout)

The user must be authenticated before entering checkout. Use OTP login with `register_if_not_exists: true` for a seamless flow — new users are registered automatically, existing users log in.

See `auth/` skill for the full OTP login implementation (phone, email, or WhatsApp).

### Step 3: Collect & Update Addresses

```typescript
// Fetch saved addresses
const { data: addresses } = await sdk.customer.listAddresses({ user_id: userId });

// Update cart with selected address IDs
const { data, error } = await sdk.cart.updateCartAddress(
  { id: cartId },
  {
    billing_address_id: "addr_billing_123",
    shipping_address_id: "addr_shipping_456",
  }
);
```

### Step 3: Validate Fulfillment

```typescript
// Check if shipping is available to the destination pincode
const { data: fulfillment, error } = await sdk.cart.checkPincodeDeliverability({
  delivery_pincode: "400001",
  cart_id: cartId,
});
// fulfillment.is_serviceable → boolean
// fulfillment.unserviceable_items → items that can't be delivered

// Get available fulfillment options (delivery, collect-in-store)
const { data: options, error: optErr } = await sdk.cart.getFulfillmentOptions({
  cart_id: cartId,
  // fulfillment_type: "delivery" — optional, omit to get all options
});

// options.summary → { collect_available, deliver_available, recommended_fulfillment_type }
// options.deliver → { shipments: [{ id, shipping_methods: [...] }] }
// options.collect → store locations for collect-in-store

// Update fulfillment preference — delivery example:
const { data: prefData, error: prefErr } = await sdk.cart.updateFulfillmentPreference(
  { id: cartId },
  {
    fulfillment_type: "delivery",
    shipments: [
      {
        id: shipmentId,
        shipping_provider_id: selectedMethodId,
        courier_company_id: selectedCourierId, // optional, if method has couriers
      },
    ],
  }
);

// For collect-in-store:
// await sdk.cart.updateFulfillmentPreference(
//   { id: cartId },
//   { fulfillment_type: "collect-in-store", pickup_location_id: storeId }
// );
```

### Step 4: Apply Discounts (Optional)

```typescript
// Apply coupon
await sdk.cart.applyCoupon({ id: cartId }, { coupon_code: "SAVE20" });

// Apply loyalty points
await sdk.cart.redeemLoyaltyPoints({ id: cartId }, { loyalty_point_redeemed: 500 });

// Remove loyalty points
await sdk.cart.removeLoyaltyPoints({ id: cartId });
```

### Step 5: Create Order & Initiate Payment

> See `payment-patterns.md` § "Building Payment Payloads" for all payment_method shapes (hyper checkout, cards, UPI, net banking, saved cards).

```typescript
const { data: order, error } = await sdk.order.createOrder({
  cart_id: cartId,
  payment_method: {
    payment_provider_slug: "juspay",
    integration_type: "hyper-checkout",
    gateway_reference_id: gatewayRefId,
    return_url: "https://yoursite.com/order/confirm",
    action: "paymentPage",
  },
});

if (order.payment_required) {
  // Use payment_info from response to proceed with payment
  // For hyper-checkout: redirect to payment_info.payment_links.web
  // For express-checkout: handle 3DS redirect or direct OTP flow
} else {
  // Order placed without payment (e.g., 100% coupon/loyalty)
  showConfirmation(order.order.order_number);
}
```

### Step 6: Handle Payment Callback

After the user returns from the payment gateway:

```typescript
// Poll payment status — note: takes a string, not an object
const { data: paymentStatus, error } = await sdk.order.getPaymentStatus(orderNumber);

switch (paymentStatus.status) {
  case "success":
    showOrderConfirmation(orderNumber);
    break;
  case "pending":
    // Still processing — poll again after delay
    setTimeout(() => pollPaymentStatus(orderNumber), 3000);
    break;
  case "failed":
    if (paymentStatus.is_retry_available) {
      showRetryPayment(orderNumber);
    } else {
      showPaymentFailed(orderNumber);
    }
    break;
}
```

### Step 7: Retry Payment (If Failed)

```typescript
if (paymentStatus.is_retry_available) {
  const { data, error } = await sdk.order.retryOrderPayment(
    { order_number: orderNumber },
    {
      payment_method: {
        payment_provider_slug: "juspay",
        integration_type: "hyper-checkout",
        gateway_reference_id: gatewayRefId,
        return_url: "https://yoursite.com/order/confirm",
        action: "paymentPage",
      },
    }
  );
  // Use payment_info from response to proceed with retry
}
```

## Visual Flow

```
Review Cart → Auth (OTP) → Set Addresses → Check Fulfillment → Apply Discounts
    → Create Order → Payment Gateway → Poll Status → Confirmation
                                          ↓
                                   Failed? → Retry Payment
```
