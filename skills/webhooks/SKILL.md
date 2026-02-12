---
name: ce-webhooks
description: Commerce Engine webhook events and syncing. 14 event types for orders, payments, refunds, and shipments. Signature verification and async processing patterns.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.0.0"
---

# Webhooks & Events

> **Prerequisite**: Webhooks are asynchronous. Use for background tasks (sync, notifications), not synchronous flows.

## Quick Reference

| Task | Action |
|------|--------|
| 1. Create endpoint | `/api/webhooks/ce` route in your app |
| 2. Verify signature | Validate webhook signature header |
| 3. Process event | Route by `event_type`, queue heavy work |
| 4. Return 200 | Respond quickly to acknowledge receipt |

## Supported Events (14 Types)

| Category | Events |
|----------|--------|
| **Order** | `order.created`, `order.confirmed`, `order.completed`, `order.cancelled` |
| **Payment** | `payment.success`, `payment.failed`, `payment.retried` |
| **Refund** | `payment.refund.initiated`, `payment.refund.success`, `payment.refund.failed` |
| **Shipment** | `shipment.created`, `shipment.updated`, `shipment.delivered`, `shipment.cancelled` |

## Decision Tree

```
Webhook Event Received
    │
    ├─ Verify signature → Invalid? → Return 401
    │
    ├─ order.created → Create order record in DB
    ├─ order.confirmed → Update order status, send confirmation email
    ├─ order.completed → Mark order as fulfilled
    ├─ order.cancelled → Update status, process refund logic
    │
    ├─ payment.success → Mark order as paid
    ├─ payment.failed → Notify customer, update status
    ├─ payment.retried → Log retry attempt, update payment status
    │
    ├─ payment.refund.initiated → Log refund request
    ├─ payment.refund.success → Update order, notify customer of refund
    ├─ payment.refund.failed → Alert team, log failure
    │
    ├─ shipment.created → Log shipment, generate tracking
    ├─ shipment.updated → Update tracking info, notify customer
    ├─ shipment.delivered → Update status, trigger review prompt
    └─ shipment.cancelled → Handle cancelled shipment
```

## Key Patterns

### Webhook Endpoint (Next.js App Router)

```typescript
// app/api/webhooks/ce/route.ts
import { NextRequest, NextResponse } from "next/server";
import crypto from "crypto";

export async function POST(req: NextRequest) {
  const body = await req.text();
  const signature = req.headers.get("x-webhook-signature");

  // 1. Verify signature
  if (!verifyWebhookSignature(body, signature)) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const event = JSON.parse(body);

  // 2. Route by event type
  switch (event.event_type) {
    case "order.created":
      await handleOrderCreated(event.data);
      break;
    case "order.confirmed":
      await handleOrderConfirmed(event.data);
      break;
    case "order.completed":
      await handleOrderCompleted(event.data);
      break;
    case "payment.success":
      await handlePaymentSuccess(event.data);
      break;
    case "payment.failed":
      await handlePaymentFailed(event.data);
      break;
    case "payment.refund.success":
      await handleRefundSuccess(event.data);
      break;
    case "shipment.delivered":
      await handleShipmentDelivered(event.data);
      break;
    // ... handle other events
    default:
      console.log(`Unhandled event: ${event.event_type}`);
  }

  // 3. Return 200 quickly
  return NextResponse.json({ received: true }, { status: 200 });
}
```

### Webhook Endpoint (Express)

```typescript
import express from "express";
import crypto from "crypto";

app.post("/webhooks/ce", express.raw({ type: "application/json" }), (req, res) => {
  const signature = req.headers["x-webhook-signature"];

  if (!verifyWebhookSignature(req.body, signature)) {
    return res.status(401).json({ error: "Invalid signature" });
  }

  const event = JSON.parse(req.body.toString());

  // Queue async work — return 200 immediately
  queueWebhookProcessing(event);

  res.status(200).json({ received: true });
});
```

### Signature Verification

```typescript
function verifyWebhookSignature(
  body: string,
  signature: string | null
): boolean {
  if (!signature) return false;

  const secret = process.env.CE_WEBHOOK_SECRET!;
  const expectedSignature = crypto
    .createHmac("sha256", secret)
    .update(body)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

### Async Processing Pattern

```typescript
// Return 200 immediately, process in background
async function handleOrderCreated(data: any) {
  // For quick operations, handle inline:
  await db.orders.upsert({
    where: { order_number: data.order_number },
    update: { status: data.status },
    create: { order_number: data.order_number, ...data },
  });

  // For heavy operations, queue:
  await queue.enqueue("send-order-confirmation", {
    order_number: data.order_number,
    user_email: data.customer_email,
  });
}
```

## When to Use Webhooks

**Do use when:**
- Syncing order data to your database
- Sending notifications (email, SMS, push)
- Triggering fulfillment workflows
- Updating inventory in external systems
- Analytics and reporting pipelines

**Don't use when:**
- Need immediate response (use API polling instead)
- Building real-time UI updates (use SDK events or polling)
- Need guaranteed ordering (webhooks may arrive out of order)

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | No signature verification | Always verify `x-webhook-signature` before processing |
| CRITICAL | Webhook route requires auth | Make webhook route public — exclude from auth middleware |
| HIGH | Handler timeout | Return 200 immediately, queue heavy work for background processing |
| HIGH | Missing idempotency | Use `event.id` or `order_number` as idempotency key — webhooks may be retried |
| MEDIUM | Handling only one event | Subscribe to all relevant events (created, confirmed, cancelled, etc.) |
| MEDIUM | Exposing webhook secret | Store `CE_WEBHOOK_SECRET` in environment variables, never commit to code |

## Webhook Reliability

- **Retries**: Failed deliveries (non-2xx response) are retried with exponential backoff
- **Ordering**: Events may arrive out of order — use timestamps to resolve conflicts
- **Idempotency**: Same event may be delivered multiple times — design handlers to be idempotent

## See Also

- `orders/` - Order data structure and API
- `setup/` - Environment variable configuration

## Documentation

- **LLM Reference (Webhooks)**: https://llm-docs.commercengine.io/webhooks/
- **Node.js Integration**: https://www.commercengine.io/docs/sdk/nodejs-integration
- **AI Prompts (Webhook Handler)**: https://www.commercengine.io/docs/ai/prompts
