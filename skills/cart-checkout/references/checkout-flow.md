# Checkout Flow: Cart to Order

Step-by-step guide for converting a cart into a placed order.

## Prerequisites

- A valid `cart_id` with items added
- A valid user session token (`access_token`)

## Flow

### Step 1: Review Cart

```typescript
const { data: cart, error } = await sdk.cart.retrieveCart(cartId);

// Display: cart_items, subtotal, grand_total, applied discounts
// Check expires_at to ensure cart hasn't expired
```

### Step 2: Collect & Update Addresses

**Logged-in users** — select from saved addresses:

```typescript
// Fetch saved addresses
const { data: addresses } = await sdk.customer.listAddresses(userId);

// Update cart with selected address IDs
const { data, error } = await sdk.cart.updateCartAddress(cartId, {
  billing_address_id: "addr_billing_123",
  shipping_address_id: "addr_shipping_456",
});
```

**Anonymous / guest users** — provide full address objects:

```typescript
const { data, error } = await sdk.cart.updateCartAddress(cartId, {
  billing_address: {
    first_name: "Jane",
    last_name: "Doe",
    address_line1: "123 Main St",
    city: "Mumbai",
    state: "Maharashtra",
    pincode: "400001",
    country: "India",
    phone: "9876543210",
    country_code: "+91",
    email: "jane@example.com",
  },
  shipping_address: { /* same structure */ },
});
```

### Step 3: Validate Fulfillment

```typescript
// Check if shipping is available to the destination pincode
const { data: fulfillment, error } = await sdk.cart.checkFulfillment(cartId);

// Get available fulfillment options (delivery, click & collect)
const { data: options, error } = await sdk.cart.retrieveFulfillmentOptions(cartId);

// options.summary → { collect_available, deliver_available, recommended_fulfillment_type }
// options.deliver → delivery details
// options.collect → store locations for Click & Collect

// Update fulfillment preference
const { data, error } = await sdk.cart.updateFulfillmentPreference(cartId, {
  fulfillment_type: "deliver",  // or "collect"
});
```

### Step 4: Apply Discounts (Optional)

```typescript
// Apply coupon
await sdk.cart.applyCoupon(cartId, { coupon_code: "SAVE20" });

// Apply loyalty points
await sdk.cart.applyLoyaltyPoints(cartId, { points: 500 });

// Apply store credit
await sdk.cart.applyCreditBalance(cartId, { amount: 100 });
```

### Step 5: Create Order & Initiate Payment

```typescript
const { data: order, error } = await sdk.order.createOrder({
  cart_id: cartId,
  payment_method: "juspay",  // or other configured gateway
  return_url: "https://yoursite.com/order/confirm",
});

if (order.payment_required) {
  // Redirect user to payment gateway using payment_info
  const { payment_url } = order.payment_info;
  window.location.href = payment_url;
} else {
  // Order placed without payment (e.g., 100% coupon, credit balance)
  showConfirmation(order.order_number);
}
```

### Step 6: Handle Payment Callback

After the user returns from the payment gateway:

```typescript
// Poll payment status from your return URL
const { data: paymentStatus, error } = await sdk.order.retrievePaymentStatus(orderNumber);

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
  const { data, error } = await sdk.order.retryPayment(orderNumber);
  // Redirect to new payment URL
  window.location.href = data.payment_info.payment_url;
}
```

## Visual Flow

```
Review Cart → Set Addresses → Check Fulfillment → Apply Discounts
    → Create Order → Payment Gateway → Poll Status → Confirmation
                                          ↓
                                   Failed? → Retry Payment
```
