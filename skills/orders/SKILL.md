---
name: ce-orders
description: Commerce Engine order management - create orders, list/detail, shipment tracking, payment status, cancellation, returns, and refunds.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Order Management

> **Prerequisite**: Order creation requires a cart with items and addresses. See `cart-checkout/`.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| Create order | `sdk.order.createOrder({ cart_id, payment_method, return_url })` |
| List orders | `sdk.order.listOrders({ query: { user_id, page, limit } })` |
| Get order detail | `sdk.order.retrieveOrder(orderNumber)` |
| Get shipments | `sdk.order.retrieveOrderShipments(orderNumber)` |
| Get payments | `sdk.order.retrieveOrderPayments(orderNumber)` |
| Payment status | `sdk.order.retrievePaymentStatus(orderNumber)` |
| Retry payment | `sdk.order.retryPayment(orderNumber)` |
| Cancel order | `sdk.order.cancelOrder(orderNumber, { cancellation_reason, refund_mode })` |
| Create return | `sdk.order.createOrderReturn(orderNumber, { return_items })` |
| Get return detail | `sdk.order.retrieveOrderReturnDetail(orderNumber, returnId)` |
| Get refunds | `sdk.order.retrieveOrderRefunds(orderNumber)` |

## Decision Tree

```
User Request
    │
    ├─ "Place order" / "Checkout"
    │   └─ sdk.order.createOrder() → handle payment_info
    │       → See cart-checkout/ for full checkout flow
    │
    ├─ "My orders" / "Order history"
    │   └─ sdk.order.listOrders({ query: { user_id } })
    │
    ├─ "Order detail" / "Order #123"
    │   └─ sdk.order.retrieveOrder(orderNumber)
    │
    ├─ "Track shipment"
    │   └─ sdk.order.retrieveOrderShipments(orderNumber)
    │       → tracking_url, awb_no, status, eta_delivery
    │
    ├─ "Payment failed" / "Retry"
    │   ├─ Check → sdk.order.retrievePaymentStatus(orderNumber)
    │   └─ Retry → sdk.order.retryPayment(orderNumber)
    │
    ├─ "Cancel order"
    │   ├─ Check is_cancellation_allowed first
    │   └─ sdk.order.cancelOrder(orderNumber, { ... })
    │
    └─ "Return" / "Refund"
        └─ sdk.order.createOrderReturn(orderNumber, { return_items })
```

## Order Statuses

| Status | Description |
|--------|-------------|
| `draft` | Order created, awaiting payment |
| `confirmed` | Payment successful, order confirmed |
| `processing` | Being prepared for shipment |
| `shipped` | In transit |
| `delivered` | Delivered to customer |
| `cancelled` | Order cancelled |

## Payment Statuses

| Status | Description | Action |
|--------|-------------|--------|
| `pending` | Payment not yet processed | Poll again after delay |
| `success` | Payment successful | Show confirmation |
| `failed` | Payment failed | Check `is_retry_available` |

## Key Patterns

### Create Order from Cart

```typescript
const { data: order, error } = await sdk.order.createOrder({
  cart_id: cartId,
  payment_method: "juspay",
  return_url: "https://yoursite.com/order/confirm",
});

if (order.payment_required) {
  // Redirect to payment gateway
  window.location.href = order.payment_info.payment_url;
}
```

### List Orders with Filters

```typescript
const { data, error } = await sdk.order.listOrders({
  query: {
    user_id: userId,
    page: 1,
    limit: 10,
    status: ["confirmed", "shipped", "delivered"],
    sort_by: '{"order_date":"desc"}',
  },
});

// data.orders → OrderList[] (summary view)
// data.pagination → { total_records, total_pages, next_page }
```

### Poll Payment Status

```typescript
async function pollPaymentStatus(orderNumber: string) {
  const { data, error } = await sdk.order.retrievePaymentStatus(orderNumber);

  if (data.status === "pending") {
    setTimeout(() => pollPaymentStatus(orderNumber), 3000);
    return;
  }

  if (data.status === "success") {
    showConfirmation(orderNumber);
  } else if (data.is_retry_available) {
    showRetryOption(orderNumber);
  } else {
    showPaymentFailed();
  }
}
```

### Cancel Order

```typescript
// First check if cancellation is allowed
const { data: order } = await sdk.order.retrieveOrder(orderNumber);

if (order.is_cancellation_allowed) {
  const { data, error } = await sdk.order.cancelOrder(orderNumber, {
    cancellation_reason: "Changed my mind",
    refund_mode: "original_payment_mode",
    feedback: "Optional feedback",
  });
}
```

### Initiate Return

```typescript
const { data, error } = await sdk.order.createOrderReturn(orderNumber, {
  return_items: [
    {
      product_id: "prod_123",
      sku: "SKU-001",
      quantity: 1,
      resolution: "refund",     // or "replacement"
      return_reason: "Defective item",
    },
  ],
});

// Return requests typically require approval via Admin Portal
```

### Shipment Tracking

```typescript
const { data, error } = await sdk.order.retrieveOrderShipments(orderNumber);

data.shipments.forEach((shipment) => {
  console.log(shipment.status);        // "shipped", "delivered", etc.
  console.log(shipment.tracking_url);  // Courier tracking link
  console.log(shipment.awb_no);        // Air waybill number
  console.log(shipment.eta_delivery);  // Estimated delivery date
});
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Creating order without address | Cart must have billing/shipping addresses set before `createOrder()` |
| HIGH | Not polling payment status | After gateway redirect, always poll `retrievePaymentStatus()` |
| HIGH | Showing cancel button when not allowed | Check `is_cancellation_allowed` from order detail first |
| MEDIUM | Not handling multiple shipments | One order can have multiple shipments — iterate `data.shipments` |
| MEDIUM | Missing `return_url` | Must provide `return_url` for payment gateway callback |
| LOW | Not filtering orders by status | Use `status[]` query param to show relevant orders per tab |

## See Also

- `cart-checkout/` - Cart management and checkout flow
- `webhooks/` - Real-time order status updates via webhooks
- `subscriptions/` - Recurring orders managed separately

## Documentation

- **Checkout Flow**: https://www.commercengine.io/docs/storefront/checkout
- **My Account - Orders**: https://www.commercengine.io/docs/storefront/my-account
- **API Reference**: https://www.commercengine.io/docs/api-reference/orders
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/create-order
