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
// Fetch saved addresses (customer_id is resolved from the session)
const { data: addresses } = await sdk.customer.listAddresses();

// Update cart with selected address IDs
const { data, error, events } = await sdk.cart.updateCartAddress(
  { id: cartId },
  {
    billing_address_id: "addr_billing_123",
    shipping_address_id: "addr_shipping_456",
  }
);
// events: `Checkout Started`, on every address update (even an unchanged one)
```

### Step 4: Validate Fulfillment

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

### Step 5: Apply Discounts (Optional)

```typescript
// Apply coupon (events: `Coupon Applied`)
await sdk.cart.applyCoupon({ id: cartId }, { coupon_code: "SAVE20" });

// What the customer can redeem
const { data: program } = await sdk.customer.getLoyaltyProgram(); // 404 when the store has no loyalty program
const { data: loyalty } = await sdk.customer.getLoyaltyDetails();
const { data: credit } = await sdk.customer.getCreditBalance();
// program.loyalty_program.point_value, loyalty.loyalty.points_balance, credit.balance_amount (spendable)

// Apply loyalty points
await sdk.cart.redeemLoyaltyPoints({ id: cartId }, { loyalty_point_redeemed: 500 });

// Remove loyalty points
await sdk.cart.removeLoyaltyPoints({ id: cartId });

// Apply credit balance (sets the amount used), or remove it
await sdk.cart.redeemCreditBalance({ id: cartId }, { credit_balance_used: 250 });
await sdk.cart.removeCreditBalance({ id: cartId });
```

### Step 6: Create Order & Initiate Payment

> See `payment-patterns.md` § "Building Payment Payloads" for all payment_method shapes (hyper checkout, cards, UPI, net banking, saved cards).

```typescript
const { data: order, error, events } = await sdk.order.createOrder({
  cart_id: cartId,
  payment_method: {
    payment_provider_slug: "juspay",
    integration_type: "hyper-checkout",
    gateway_reference_id: gatewayRefId,
    return_url: "https://yoursite.com/order/confirm",
    action: "paymentPage",
  },
});
// events: `Order Created` — forward it if you track in the browser (see `analytics/`)

if (error) {
  // 409 `credit-balance-unavailable`: the credit balance can't be used right now. Nothing was placed; retry shortly
  return showError(error.message);
}

if (order.payment_required && order.payment_info) {
  // Use payment_info from response to proceed with payment
  // For hyper-checkout: redirect to payment_info.payment_links.web
  // For express-checkout: handle 3DS redirect or direct OTP flow
} else {
  // No gateway payment started: nothing left to pay (coupon, loyalty, credit balance),
  // a credit-line order, a manual method or a bank transfer. Poll getPaymentStatus for the outcome.
  showConfirmation(order.order.order_number);
}
```

### Step 7: Handle Payment Callback

After the user returns from the payment gateway:

```typescript
// Poll payment status — note: takes a string, not an object
const { data: paymentStatus, error } = await sdk.order.getPaymentStatus(orderNumber);

switch (paymentStatus.status) {
  case "success":
    showOrderConfirmation(orderNumber);
    break;
  case "pending":
  case "partially_paid":
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

### Step 8: Retry Payment (If Failed)

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
  // Use data.payment_info to proceed (null for a manual method or bank transfer)
}
```

To pay only part of what is left with another method, or split the rest across methods, use `sdk.order.createOrderPaymentRequest({ order_number }, { amount, payment_method })` instead. To switch methods while an express-checkout payment is still pending, cancel it first with `sdk.order.cancelPaymentRequest({ order_number })`. See `payment-patterns.md` § "Retry Payment".

## Visual Flow

```
Review Cart → Auth (OTP) → Set Addresses → Check Fulfillment → Apply Discounts
    → Create Order → Payment Gateway → Poll Status → Confirmation
                                          ↓
                                   Failed? → Retry Payment (or pay the rest with another method)
```
