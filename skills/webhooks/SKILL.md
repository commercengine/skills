---
name: ce-webhooks
description: Commerce Engine webhooks - the signed WebhookEnvelope, X-CE-* headers, sha256 HMAC signature verification, event_id idempotency, retries and 410, and the 241-event catalogue across orders, payments, shipping, invoices, carts, coupons, catalog, customers, store and marketplace.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.1.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Webhooks & Events

> **Prerequisite**: Webhooks are asynchronous. Use them for background work (sync, notifications, fulfilment), not for synchronous flows.

Commerce Engine sends every event a webhook subscribes to as a signed `POST` with a JSON `WebhookEnvelope` body. The contract is the published webhook spec ([`ce-webhooks.json`](https://openapi.commercengine.io/ce-webhooks.json), OpenAPI 3.2), and the [webhook index](https://llm-docs.commercengine.io/webhooks/) documents every event's payload.

## Quick Reference

| Step | Action |
|------|--------|
| 1. Register | Register the endpoint URL and the events it subscribes to through the admin Webhooks API. Keep the webhook's secret in `CE_WEBHOOK_SECRET` |
| 2. Read the raw body | Verify the exact bytes received, before any JSON parsing |
| 3. Verify | `X-CE-Signature` is `sha256=` + hex HMAC-SHA256 of the raw body, keyed with the secret. Compare in constant time; reject with `401` |
| 4. Deduplicate | Key on `event_id` (`X-CE-Event-ID`): it's the same on every retry. `delivery_id` changes per attempt |
| 5. Route | `switch (envelope.event)`. The record is in `payload.properties`; entity ids sit on `payload` (`payload.order_number`) |
| 6. Acknowledge | Return `2XX` promptly and queue heavy work. Failed deliveries are retried 3 times (5s, 30s, 5m); a `410` disables the webhook |

## Delivery Contract

### Headers

Every request carries four headers. The last three repeat body fields, so you can log or route before parsing, but trust them only after the signature checks out.

| Header | Equals | Use it for |
|--------|--------|------------|
| `X-CE-Signature` | - | `sha256=` followed by 64 lowercase hex characters (`^sha256=[0-9a-f]{64}$`): the HMAC-SHA256 of the raw request body, keyed with the webhook secret |
| `X-CE-Event` | body `event` | Event name, e.g. `order.created` |
| `X-CE-Event-ID` | body `event_id` | Idempotency key. Stable across retries of the same event |
| `X-CE-Delivery-ID` | body `delivery_id` | Unique per delivery attempt. Use it to trace attempts, not to deduplicate |

Header names are case-insensitive. Node.js lowercases them (`req.headers["x-ce-signature"]`), and `Headers.get()` and Express `req.get()` match any case.

### Envelope (`WebhookEnvelope`)

| Field | Type | Meaning |
|-------|------|---------|
| `event` | string | Event name, e.g. `order.created` |
| `event_id` | string (ULID) | Shared by every delivery attempt of this event |
| `delivery_id` | string (ULID) | Unique to this delivery attempt |
| `timestamp` | date-time | When this delivery attempt was made |
| `is_test` | boolean | `true` when sent from the admin *Send test webhook* action |
| `payload` | `WebhookPayload` | The event record (below) |

### Event record (`payload`)

| Field | Meaning |
|-------|---------|
| `event`, `event_id` | Identical to the envelope's |
| `store_id` | Store the event belongs to. For `seller.*` events this is the marketplace store; the seller store is in `seller_id` |
| `store_type` | `b2c`, `b2b`, `marketplace` or `seller` (seller-side copies of marketplace records); may be `null` |
| Entity ids | Each event lists its own at the top level: its entity's (`order_number`, `shipment_id`, `cart_id`, ...) and related ones (an order event's `cart_id`, a refund's `order_number`) |
| `user_id` | The storefront user the event concerns: the shopper who caused it, or whose record an admin or the system changed |
| `customer_id` | Present on customer, KYC, loyalty and invoice events |
| `admin_user_id` | The admin user who caused the event, when it came from the admin API |
| `timestamp` | When the event was produced (ISO 8601, UTC). Absent on a few events |
| `changes` | `{ field: { from, to } }` for the fields of `properties` that changed (and, on some events, details of the action). `{}` when there's nothing to list |
| `context` | `channel`, `channel_id`, `ip`, `user_agent`, `locale`, `anonymous_id`, `session_id`, `page`, `campaign`. Delivered only to webhooks that opt in; ad identifiers only with the shopper's marketing consent |
| `properties` | The event-specific model: `OrderState`, `OrderPaymentEvent`, `OrderRefundEvent`, `ShipmentList` and others. Each event's page names it |

```json
{
  "event": "order.confirmed",
  "event_id": "01JD3Q6W5X8K2M9N4P7R1T0V3Y",
  "delivery_id": "01JD3Q6W9B4C7D2E5F8G1H3J6K",
  "timestamp": "2026-11-15T09:30:12.481Z",
  "is_test": false,
  "payload": {
    "event": "order.confirmed",
    "event_id": "01JD3Q6W5X8K2M9N4P7R1T0V3Y",
    "store_id": "01J9ZQ2B7K4M8N3P6R1T5V9W0X",
    "store_type": "b2c",
    "order_number": "ORD-1001",
    "timestamp": "2026-11-15T09:30:11.902Z",
    "changes": { "status": { "from": "pending", "to": "confirmed" } },
    "properties": { "order_number": "ORD-1001", "status": "confirmed", "payment_status": "success", "grand_total": 1499 }
  }
}
```

*Illustrative values; `properties` is trimmed. [`OrderState`](https://llm-docs.commercengine.io/events/schemas/OrderState) has the full order.*

### Responses and Retries

| Your response | What happens |
|---------------|--------------|
| `2XX` | Accepted. The delivery is done |
| `410 Gone` | Read as "endpoint permanently removed": the webhook is **disabled** and gets no more deliveries |
| `5XX`, or any other failure | A failed delivery, retried three times, after 5s, 30s and 5m. Each retry has the same `event_id`, with a new `delivery_id` and envelope `timestamp` |

- Respond promptly. The spec doesn't publish a timeout, so acknowledge first and do the work in the background.
- The spec lists no attempts after the third retry. After an outage longer than that window, re-read anything critical from the Commerce Engine API.
- Return `2XX` for events you don't handle. A `4XX` is a failed delivery and gets retried; a `410` turns the webhook off.

### Ordering

The spec makes no ordering guarantee, and retries mean an older event can arrive after a newer one.
- `payload.timestamp` is when the event was produced, but a few events don't carry it. The spec says to prefer the envelope `timestamp` for ordering. That's the time of the attempt, so a retry carries a later time than its first attempt.
- Write state so a late arrival can't roll it back. Apply a record only if it's newer than what you hold (compare `payload.timestamp` where it's present), or re-read it from the API when the latest state matters.

## Event Catalogue (241 Events)

Grouped as the spec tags them. The [webhook index](https://llm-docs.commercengine.io/webhooks/) is the live list, and each event has a page at `https://llm-docs.commercengine.io/events/webhooks/{event}` (e.g. [`checkout.completed`](https://llm-docs.commercengine.io/events/webhooks/checkout.completed)) with its description, ids and `properties` model.

| Section | Tag | Events | Representative events | Ids on `payload` |
|---------|-----|-------:|-----------------------|------------------|
| Orders | Orders (order lifecycle) | 13 | `order.created`, `checkout.completed`, `order.confirmed`, `order.awaiting_approval`, `order.payment_failed`, `order.cancelled`, `order.completed`, `order.updated`, `order.return.requested` | `order_number` (returns: `return_request_number`) |
| | Payments (payments and refunds) | 28 | `payment.created`, `payment.success`, `payment.failed`, `payment.retried`, `payment.refund.initiated` / `.success` / `.failed`, `payment.settled`, `payment.exception.recorded`, `credit_balance.updated` | `payment_request_number`, `refund_request_number` or `payment_id`, plus `order_number` when there is one |
| | Shipping | 10 | `shipment.created`, `shipment.scheduled`, `shipment.shipped`, `shipment.delivered`, `shipment.returned`, `shipment.cancelled`, `shipment.updated` | `order_number`, `shipment_id` |
| | Invoices | 6 | `invoice.created`, `invoice.updated`, `external_invoice.created`, `external_invoice.cancelled` | `invoice_id` |
| Shopping | Carts (carts and wishlists) | 15 | `cart.created`, `cart.item.created` / `.updated` / `.deleted`, `cart.coupon.applied`, `cart.address.updated`, `wishlist.item.created` | `cart_id` (wishlists: `user_id`) |
| | Coupons & promotions | 6 | `coupon.created` / `.updated` / `.deleted`, `promotion.created` / `.updated` / `.deleted` | `coupon_id`, `promotion_id` |
| Catalog | Catalog (products, variants, categories, attributes, inventory) | 23 | `product.created` / `.updated` / `.deleted`, `product.variant.*`, `inventory.updated`, `category.*`, `product.review.*` | `product_id`, `variant_id`, `category_id` |
| Customers | Customers (customers, addresses, KYC, loyalty) | 58 | `customer.created` / `.updated`, `user.account.created`, `customer.address.*`, `kyc.verified`, `loyalty.points.earned`, `tier.upgraded`, `customer.mandate.*` (NACH), `newsletter.subscribed` | `customer_id` or `user_id` |
| Store | Store (configuration) | 25 | `store.config.updated`, `channel.*`, `warehouse.*`, `shipping_zone.*`, `store.shipping_method.*`, `store.customer_group.*` | The entity's id (`channel_id`, `warehouse_id`, ...) |
| | Marketplace (marketplace and seller store sync) | 57 | `seller.order.*`, `seller.account.*`, `seller.kyc.*`, `seller.payout.batch.*`, `marketplace_item.*`, `product.marketplace-listing.*`, `product.marketplace-sync.*` | `seller_id`, `order_number`, `product_id`, ... |

## Decision Tree

```
Webhook request received (POST)
    │
    ├─ X-CE-Signature missing or wrong → 401, act on nothing
    ├─ is_test: true → 200, no side effects (admin "Send test webhook")
    ├─ event_id already processed → 200 (a retry; do nothing)
    │
    ├─ Orders: payload.order_number, properties = OrderState
    │   ├─ order.created → record the order (placed; may not be paid yet)
    │   ├─ checkout.completed → purchased (once per order): confirmation email, conversion
    │   ├─ order.confirmed → ready to fulfil (auto-approved, admin-accepted or settled)
    │   ├─ order.awaiting_approval → paid, waiting for the merchant to approve it
    │   ├─ order.payment_failed → every payment of a pending order failed: nudge the shopper
    │   ├─ order.cancelled → update status; refunds follow as payment.refund.*
    │   └─ order.completed → every shipment delivered (or a collect-in-store order paid)
    │
    ├─ Payments: per payment attempt (payload.payment_request_number + order_number)
    │   ├─ payment.success / payment.failed / payment.retried → update the attempt
    │   └─ payment.refund.initiated → .success | .failed (payload.refund_request_number)
    │
    ├─ Shipping: payload.order_number + shipment_id, properties = ShipmentList
    │   ├─ shipment.created / scheduled / shipped / delivered / returned / cancelled
    │   └─ shipment.updated → admin changes, incl. manual shipped/delivered: read properties.status
    │
    ├─ Catalog: product.* / product.variant.* / inventory.updated → refresh caches and search
    ├─ Customers: customer.* / user.account.* / loyalty.* → sync your CRM
    │
    └─ Anything else → 200 and ignore (a non-2XX is retried; a 410 disables the webhook)
```

## Key Patterns

### Verify and Queue (Framework-Agnostic)

Takes the raw body and a header getter, and returns the status to send. Uses only `node:crypto`, so it runs in any Node.js framework.

```typescript
// lib/ce-webhook.ts
import { createHmac, timingSafeEqual } from "node:crypto";

/** Body of every Commerce Engine webhook request (`WebhookEnvelope`). */
export interface CeWebhookEnvelope<P = Record<string, unknown>> {
  event: string; // e.g. "order.created"; also the X-CE-Event header
  event_id: string; // ULID, the same on every retry; also X-CE-Event-ID
  delivery_id: string; // ULID, new on every attempt; also X-CE-Delivery-ID
  timestamp: string; // ISO 8601: when this delivery attempt was made
  is_test: boolean; // true when sent from the admin "Send test webhook" action
  payload: {
    event: string;
    event_id: string;
    store_id: string;
    store_type?: string | null; // "b2c" | "b2b" | "marketplace" | "seller"
    user_id?: string | null;
    customer_id?: string | null;
    admin_user_id?: string | null;
    seller_id?: string | null; // seller.* events only
    timestamp?: string; // when the event was produced; absent on a few events
    changes?: Record<string, { from?: unknown; to?: unknown }>;
    context?: Record<string, unknown>; // only for webhooks that opt in
    properties: P; // the event's model: see its page in the webhook index
    [id: string]: unknown; // entity ids: order_number, shipment_id, cart_id, ...
  };
}

export function verifyCeSignature(
  rawBody: Buffer | string,
  header: string | null | undefined,
  secret: string,
): boolean {
  if (!header?.startsWith("sha256=")) return false;
  const received = Buffer.from(header.slice("sha256=".length), "utf8");
  const expected = Buffer.from(
    createHmac("sha256", secret).update(rawBody).digest("hex"),
    "utf8",
  );
  // timingSafeEqual throws on a length mismatch, so check the length first
  return received.length === expected.length && timingSafeEqual(received, expected);
}

/** Returns the HTTP status to send. Throws (→ 5XX, retried) when it can't take the event. */
export async function handleCeWebhook(
  rawBody: Buffer | string,
  getHeader: (name: string) => string | null | undefined,
): Promise<number> {
  const secret = process.env.CE_WEBHOOK_SECRET;
  if (!secret) throw new Error("CE_WEBHOOK_SECRET is not set");

  // 1. Authenticate the exact bytes received, before parsing anything
  if (!verifyCeSignature(rawBody, getHeader("x-ce-signature"), secret)) return 401;

  const envelope = JSON.parse(rawBody.toString()) as CeWebhookEnvelope;

  // 2. Sent from the admin "Send test webhook" action: acknowledge, no side effects
  if (envelope.is_test) return 200;

  // 3. Queue it keyed by event_id, so a retry of the same event is a no-op.
  //    Do the work in a worker, not in the request.
  await enqueueOnce(envelope.event_id, envelope); // your queue

  // 4. Acknowledge promptly. Never send 410 unless you want the webhook disabled.
  return 200;
}
```

### Webhook Endpoint (Next.js App Router)

```typescript
// app/api/webhooks/ce/route.ts (Node.js runtime, the default)
import { handleCeWebhook } from "@/lib/ce-webhook";

export async function POST(req: Request) {
  // Exact bytes as sent: don't call req.json() before verifying
  const rawBody = Buffer.from(await req.arrayBuffer());
  const status = await handleCeWebhook(rawBody, (name) => req.headers.get(name));
  return new Response(null, { status });
}
```

### Webhook Endpoint (Express)

```typescript
import express from "express";
import { handleCeWebhook } from "./lib/ce-webhook";

const app = express();

// Register before app.use(express.json()) so this route gets the raw bytes
app.post("/webhooks/ce", express.raw({ type: "application/json" }), async (req, res, next) => {
  try {
    const status = await handleCeWebhook(req.body as Buffer, (name) => req.get(name));
    res.sendStatus(status);
  } catch (err) {
    next(err); // 500: Commerce Engine retries the delivery
  }
});

app.use(express.json()); // every other route
```

### Idempotent Processing (Worker)

Record `event_id` in the same transaction as the side effects, so a duplicate can't slip in between the check and the write.

```typescript
import type { CeWebhookEnvelope } from "./lib/ce-webhook";

export async function processWebhook(envelope: CeWebhookEnvelope) {
  await db.transaction(async (tx) => {
    // Unique index on event_id: a second delivery of the same event inserts nothing
    if (!(await tx.processedWebhookEvents.insertIfAbsent({ event_id: envelope.event_id }))) return;
    await routeEvent(envelope, tx);
  });
}

async function routeEvent({ event, payload }: CeWebhookEnvelope, tx: Tx) {
  const id = (key: string) => String(payload[key]); // ids sit at the top of payload

  switch (event) {
    case "checkout.completed": // once per order, when it's purchased
      return sendOrderConfirmation(id("order_number"), tx);
    case "order.confirmed": // ready to fulfil
      return startFulfilment(id("order_number"), tx);
    case "order.created":
    case "order.updated":
    case "order.cancelled":
    case "order.completed":
      return saveOrder(id("order_number"), payload.properties, tx); // OrderState
    case "payment.refund.success":
    case "payment.refund.failed":
      return saveRefund(id("refund_request_number"), payload.properties, tx); // OrderRefundEvent
    case "shipment.delivered":
    case "shipment.updated": {
      // A manual "delivered" update arrives as shipment.updated
      const shipment = payload.properties as { status?: string };
      if (shipment.status === "delivered") return markShipmentDelivered(id("shipment_id"), tx);
      return;
    }
    default:
      return; // not handled here; it was still acknowledged with 2XX
  }
}
```

`db`, `Tx`, `enqueueOnce` and the handler functions (`saveOrder`, `startFulfilment`, ...) are your app's own code.

### Test Locally

The admin *Send test webhook* action sends a real signed request with `is_test: true`. To call your endpoint by hand, sign the body yourself:

```bash
BODY='{"event":"order.created","event_id":"01JD3Q6W5X8K2M9N4P7R1T0V3Y","delivery_id":"01JD3Q6W9B4C7D2E5F8G1H3J6K","timestamp":"2026-11-15T09:30:12.481Z","is_test":false,"payload":{"event":"order.created","event_id":"01JD3Q6W5X8K2M9N4P7R1T0V3Y","store_id":"01J9ZQ2B7K4M8N3P6R1T5V9W0X","order_number":"ORD-1001","properties":{}}}'
SIG="sha256=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$CE_WEBHOOK_SECRET" | sed 's/^.* //')"

curl -X POST http://localhost:3000/api/webhooks/ce \
  -H "Content-Type: application/json" \
  -H "X-CE-Event: order.created" \
  -H "X-CE-Event-ID: 01JD3Q6W5X8K2M9N4P7R1T0V3Y" \
  -H "X-CE-Delivery-ID: 01JD3Q6W9B4C7D2E5F8G1H3J6K" \
  -H "X-CE-Signature: $SIG" \
  --data-binary "$BODY"
```

Use `--data-binary` and `printf '%s'`: a trailing newline or a re-encoded body changes the signature.

## When to Use Webhooks

**Do use when:**
- Syncing orders, payments, refunds and shipments to your database
- Sending notifications (email, SMS, push)
- Triggering fulfilment, ERP or accounting workflows
- Keeping search indexes, caches and external inventory in step with the catalog
- Syncing customers, KYC and loyalty to a CRM

**Don't use when:**
- You need the result right away: use the SDK response (or poll the API)
- You're building real-time UI updates: use the SDK response or polling
- You need guaranteed ordering: deliveries can arrive out of order
- You're tracking shopper behaviour for analytics or ads: see `analytics/`

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | No signature verification | Verify `X-CE-Signature` before acting on anything in the request |
| CRITICAL | Verifying parsed or re-serialized JSON | `JSON.stringify(JSON.parse(body))` isn't the signed bytes. Read the raw body (`req.arrayBuffer()`, `express.raw()`) and register the route before any JSON body parser |
| CRITICAL | Comparing signatures with `===` | Leaks timing. Use `crypto.timingSafeEqual` after a length check |
| CRITICAL | Webhook route requires auth | Make the route public and exclude it from auth middleware. The signature is the authentication |
| HIGH | Old signature format | The header is `X-CE-Signature: sha256=<hex>`, not bare hex in `x-webhook-signature`. Strip the `sha256=` prefix before comparing |
| HIGH | Returning `410` for unknown events or missing records | `410` disables the whole webhook. Return `2XX` for events you ignore |
| HIGH | Deduplicating on `delivery_id` | It changes on every attempt. Use `event_id` (`X-CE-Event-ID`) |
| HIGH | Slow handler | Acknowledge with `2XX` first and queue the work. A failed delivery gets only three retries (5s, 30s, 5m) |
| HIGH | Routing on `event_type` or reading data at the top level | Route on `event`. The record is in `payload.properties`, and ids are on `payload` (`payload.order_number`) |
| HIGH | Waiting only for `shipment.shipped` / `shipment.delivered` | Manual status changes arrive as `shipment.updated`. Check `properties.status` |
| MEDIUM | Treating `payment.success` as "order paid" | It's per payment attempt, and an order can take several (split payments, bank-transfer settlements). `checkout.completed` fires once per order when it's purchased |
| MEDIUM | Assuming arrival order | No ordering guarantee. Don't let an older event overwrite newer state |
| MEDIUM | Acting on test sends | Check `is_test` and skip real side effects |
| MEDIUM | Subscribing to too few events | Subscribe to every event your flow depends on (e.g. `order.cancelled` and `payment.refund.*` alongside `checkout.completed`) |
| MEDIUM | Exposing the webhook secret | Keep `CE_WEBHOOK_SECRET` in environment variables and never commit it |
| LOW | Expecting `context` on every webhook | `context` is delivered only to webhooks that opt in |

## See Also

- `orders/` - Order, payment, refund and shipment data from the storefront SDK
- `cart-checkout/` - The checkout flow behind `checkout.completed` and `payment.*` events
- `analytics/` - Browser and server analytics events (`@commercengine/analytics`), separate from webhooks
- `setup/` - Environment variable configuration

## Documentation

- **Webhook index (all events)**: https://llm-docs.commercengine.io/webhooks/
- **One event**: https://llm-docs.commercengine.io/events/webhooks/{event} (e.g. `/events/webhooks/order.created`)
- **Envelope and payload**: https://llm-docs.commercengine.io/events/schemas/WebhookEnvelope, https://llm-docs.commercengine.io/events/schemas/WebhookPayload
- **Signature header**: https://llm-docs.commercengine.io/events/parameters/WebhookSignatureHeader
- **Webhook spec (OpenAPI 3.2)**: https://openapi.commercengine.io/ce-webhooks.json
