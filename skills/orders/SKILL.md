---
name: ce-orders
description: Commerce Engine order management - create orders, list/detail, shipment tracking, payment status and retries, cancellation, returns, and refunds.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.1.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Order Management

> **Prerequisite**: Order creation requires a cart with items and addresses. See `cart-checkout/`.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| Create order | `sdk.order.createOrder({ cart_id, payment_method? })` — also returns `events` |
| List orders | `sdk.order.listOrders({ page, limit })` |
| Get order detail | `sdk.order.getOrderDetails({ order_number })` |
| Get shipments | `sdk.order.listOrderShipments({ order_number })` |
| Get payments | `sdk.order.listOrderPayments({ order_number })` |
| Payment status | `sdk.order.getPaymentStatus(orderNumber)` — **takes a plain string, not `{ order_number }`** |
| Retry payment | `sdk.order.retryOrderPayment({ order_number }, { payment_method })` |
| Cancel pending payment | `sdk.order.cancelPaymentRequest({ order_number })` — to switch methods mid-payment |
| Pay the rest / split payment | `sdk.order.createOrderPaymentRequest({ order_number }, { amount, payment_method })` |
| Cancel order | `sdk.order.cancelOrder({ order_number }, { cancellation_reason, refund_mode })` — also returns `events` |
| Get refunds | `sdk.order.listOrderRefunds({ order_number })` |
| List returns | `sdk.order.listOrderReturns({ order_number })` |
| Request return / replacement | `sdk.order.createOrderReturn({ order_number }, { requested_settlement, items })` |
| Return detail | `sdk.order.getOrderReturnDetails({ order_number, return_number })` |
| Upload return photos/videos | `sdk.order.uploadOrderReturnAttachments({ order_number, return_number }, { images, videos })` |
| Cancel return | `sdk.order.cancelOrderReturn({ order_number, return_number }, { reason })` |

> **Note**: Customers can **cancel** an order within the cancellation window (`is_cancellation_allowed`), and **request a return or replacement** of delivered items on shipments with `is_return_allowed: true`. A return request starts `created`; the store approves or rejects it, and its detail links the return order, the replacement order and any refunds.

> **`events`**: `createOrder` and `cancelOrder` return `ApiResult<T, StorefrontResponseEvent>`: `{ data, error, events }`. `events` holds `Order Created` / `Order Cancelled` with a server-issued `event_id` (empty on an error, or when a cancel changed nothing). To track these in the browser, forward `events` to `@commercengine/analytics` — see `analytics/`. Test doubles for these methods must include `events` (for example `events: []`).

## Decision Tree

```
User Request
    │
    ├─ "Place order" / "Checkout"
    │   └─ sdk.order.createOrder() → handle payment_info
    │       → See cart-checkout/ for full checkout flow
    │
    ├─ "My orders" / "Order history"
    │   └─ sdk.order.listOrders({ page, limit })
    │
    ├─ "Order detail" / "Order #123"
    │   └─ sdk.order.getOrderDetails({ order_number })
    │
    ├─ "Track shipment"
    │   └─ sdk.order.listOrderShipments({ order_number })
    │       → tracking_url, awb_no, status, eta_delivery
    │
    ├─ "Payment failed" / "Retry" / "Pay the rest"
    │   ├─ Check → sdk.order.getPaymentStatus(orderNumber)
    │   ├─ Failed + is_retry_available → sdk.order.retryOrderPayment({ order_number }, { payment_method })
    │   ├─ Switch method while pending → sdk.order.cancelPaymentRequest({ order_number }), then retry
    │   └─ Part payment / split across methods → sdk.order.createOrderPaymentRequest({ order_number }, { amount, payment_method })
    │
    ├─ "Cancel order"
    │   ├─ Check is_cancellation_allowed first
    │   └─ sdk.order.cancelOrder({ order_number }, { ... })
    │
    └─ "Return" / "Replace item"
        ├─ Check the shipment's is_return_allowed first
        └─ sdk.order.createOrderReturn({ order_number }, { requested_settlement, items })
```

## Order Statuses

`order.status` (on `Order` / `OrderDetail`):

| Status | Description |
|--------|-------------|
| `pending` | Order placed, awaiting payment |
| `payment_failed` | Every payment attempt failed; a payment retry reopens it |
| `awaiting_approval` | Paid, waiting for the store to approve (stores that don't approve orders automatically) |
| `confirmed` | Paid (or accepted on the customer's credit line) and confirmed |
| `partially_scheduled`, `schedule_requested`, `awaiting_shipment`, `processing` | Being prepared for shipment |
| `shipped`, `partially_shipped`, `partially_delivered` | In transit, or partly delivered |
| `sent_to_store`, `ready_to_collect` | Collect-in-store progress |
| `complete` | Fulfilled. A collect-in-store order with nothing to ship is `complete` as soon as it is paid |
| `undelivered`, `partially_undelivered`, `return_to_origin`, `partially_return_to_origin`, `lost` | Delivery problems |
| `cancelled` | Order cancelled |

`draft` is also in the enum. There is no `delivered` order status: `delivered` is a shipment status (`shipment.status`). A `switch` that covers every value needs all of the above.

## Payment Statuses

`getPaymentStatus(orderNumber)` → `data.status`:

| Status | Description | Action |
|--------|-------------|--------|
| `pending` | Payment not yet processed | Poll again after delay |
| `partially_paid` | Part of a bank transfer or credit balance is settled | Poll again, or show as pending |
| `success` | Payment successful (stays `success` once refunded) | Show confirmation |
| `failed` | Payment failed | Check `is_retry_available` |

`order.payment_status` (on the order itself) can also be `refund_initiated` / `refunded` (refunds covering what was paid are on their way, or done), `debit_scheduled` / `collection_due` (on the customer's credit line: to be debited by NACH mandate, or collected later) and `manual_review`. Payment records from `listOrderPayments` add `initiated`, `settled`, `unsettled` and `partially_settled`.

## Refund Statuses

`listOrderRefunds` → `data.refunds[]`:

| `status` | Meaning |
|----------|---------|
| `pending` | Created, not sent yet (a `manual` refund stays `pending` until the store marks it paid) |
| `refund_initiated` | Being sent to the gateway or bank |
| `initiated` | Accepted by the gateway or bank; result awaited |
| `manual_review` | Held by the gateway for review |
| `success` / `failed` | Money returned / not returned |
| `forfeited` | The store kept money it owed back on a `manual` refund (reason in `refund_remarks`) |

`payment_mode` is `original-payment-mode` (back through the gateway), `bank-transfer`, `manual` (handed back by the store the way it was paid, such as cash) or `loyalty-point`. `payment_method`, `payment_reference_number`, `bank_account_number`, `bank_name` and `refund_remarks` can be `null`.

## Key Patterns

### Create Order from Cart

> See `cart-checkout/references/payment-patterns.md` § "Building Payment Payloads" for all payment_method shapes.

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
// events: `Order Created` → forward to @commercengine/analytics if you track in the browser

if (!error && order.payment_required && order.payment_info) {
  // Use payment_info to redirect or handle payment flow.
  // payment_info is null when no gateway payment starts (nothing to pay, credit line, manual method, bank transfer)
}
```

### List Orders with Filters

```typescript
const { data, error } = await sdk.order.listOrders({
  page: 1,
  limit: 10,
  sort_by: JSON.stringify({ order_date: "desc" }),
});

// data.orders → OrderList[] (summary view)
// data.pagination → { total_records, total_pages, next_page }
```

Order items (`order.order_items` on the detail) have no `slug` since 0.16: use `product_slug`, which can be `null`. `product_name`, `product_type`, `product_image_url` and `price_including_tax` can be `null` too.

### Poll Payment Status

```typescript
async function pollPaymentStatus(orderNumber: string, attempt = 0) {
  // Note: getPaymentStatus takes a string, not an object
  const { data, error } = await sdk.order.getPaymentStatus(orderNumber);
  if (error) return showPaymentPending(orderNumber);

  if (data.status === "pending" || data.status === "partially_paid") {
    // Gateway payments resolve in seconds; bank transfers and credit balance
    // can stay pending much longer, so cap the polling
    if (attempt < 40) setTimeout(() => pollPaymentStatus(orderNumber, attempt + 1), 3000);
    else showPaymentPending(orderNumber);
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

### Retry or Complete a Payment

```typescript
// Failed and retry available → a new payment request for the order
const { data, error } = await sdk.order.retryOrderPayment(
  { order_number: orderNumber },
  { payment_method: paymentMethod } // same shapes as createOrder
);
// data.payment_info is null for a manual method or a bank transfer

// Or pay part of what is left with another method (split payment)
const { data: part } = await sdk.order.createOrderPaymentRequest(
  { order_number: orderNumber },
  { amount: 500, payment_method: paymentMethod }
);
// part.pending_amount → what can still be requested; part.payments → all payment records
```

`createOrderPaymentRequest` can't exceed what is left of `to_be_paid` (failed and refunded payments don't count against it), takes Juspay or a manual method the store offers (not e-collection, NACH or credit line), and refuses a cancelled order (`order-cancelled`). See `cart-checkout/references/payment-patterns.md` § "Retry Payment".

### Cancel Order

```typescript
// First check if cancellation is allowed
const { data } = await sdk.order.getOrderDetails({
  order_number: orderNumber,
});
const order = data?.order;

if (order?.is_cancellation_allowed) {
  // order.cancellation_refund_details → refundable_amount, cancellation_charges, payment_mode
  const { data: cancelled, error, events } = await sdk.order.cancelOrder(
    { order_number: orderNumber },
    {
      cancellation_reason: "Changed my mind",
      refund_mode: "original-payment-mode", // or "bank-transfer"
      // bank_account_id: required when cancellation_refund_details.payment_mode is "bank-transfer"
      feedback: "Optional feedback",
    }
  );
  // events: `Order Cancelled` (none when the cancel changed nothing)
}
```

The refund follows `cancellation_refund_details.payment_mode` whatever `refund_mode` you send. A 409 means the refund can't be made right now (payment just changed, under admin review, or refunds briefly unavailable); nothing changed, so ask the customer to try again later.

### Request a Return

```typescript
const { data: shipmentData } = await sdk.order.listOrderShipments({ order_number: orderNumber });
const shipment = shipmentData?.shipments?.find((s) => s.is_return_allowed);

if (shipment?.reference_number) {
  const { data, error } = await sdk.order.createOrderReturn(
    { order_number: orderNumber },
    {
      requested_settlement: "refund", // or "replacement"
      items: [
        {
          shipment_number: shipment.reference_number, // the shipment's reference_number
          product_id: "prod_123",
          variant_id: null,
          quantity: 1,
          return_reason: "Product arrived damaged",
        },
      ],
    }
  );
  // data.order_return → { request_number, status: "created", ... }
  // Then optionally: uploadOrderReturnAttachments({ order_number, return_number: request_number }, { images, videos })
}
```

Return statuses: `created`, `approved`, `rejected`, `complete`, `cancelled`.

### Shipment Tracking

```typescript
const { data, error } = await sdk.order.listOrderShipments({
  order_number: orderNumber,
});

data?.shipments?.forEach((shipment) => {
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
| HIGH | Not polling payment status | After gateway redirect, always poll `getPaymentStatus()` |
| HIGH | Showing cancel button when not allowed | Check `is_cancellation_allowed` from order detail first |
| HIGH | `refund_mode: "original_payment_mode"` | The values are hyphenated: `"original-payment-mode"` or `"bank-transfer"`. Send `bank_account_id` when the refund goes by bank transfer |
| HIGH | Status `switch` written for older SDKs | 0.16 added order statuses (`payment_failed`, `processing`, `sent_to_store`, `ready_to_collect`, `schedule_requested`, `lost`), payment statuses (`refund_initiated`, `refunded`, `manual_review`, `collection_due`, `debit_scheduled`) and refund statuses (`refund_initiated`, `initiated`, `manual_review`, `forfeited`). Refund `status` lost `approved`, and refund `payment_mode` lost `nach-mandate` and gained `manual` |
| HIGH | Test doubles for `createOrder` / `cancelOrder` without `events` | Their result type requires `events` (use `events: []`) |
| MEDIUM | Reading `slug` on order items | Removed in 0.16 — use `product_slug` (nullable) |
| MEDIUM | Assuming `payment_info` is always set | `null` when no gateway payment starts (nothing to pay, credit line, manual method, bank transfer) |
| MEDIUM | Not handling multiple shipments | One order can have multiple shipments — iterate `data.shipments` |
| MEDIUM | Missing `return_url` | Must provide `return_url` for payment gateway callback |
| MEDIUM | Offering returns on every shipment | Only shipments with `is_return_allowed: true` can be returned; the store approves or rejects each request |
| LOW | Not filtering orders by status | Use `status[]` query param to show relevant orders per tab |

## See Also

- `cart-checkout/` - Cart management and checkout flow
- `analytics/` - Browser tracking of order events, deduplicated with `events`
- `webhooks/` - Real-time order status updates via webhooks

## Documentation

- **Checkout Flow**: https://www.commercengine.io/docs/storefront/checkout
- **My Account - Orders**: https://www.commercengine.io/docs/storefront/my-account
- **API Reference**: https://www.commercengine.io/docs/api-reference/orders
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/create-order
