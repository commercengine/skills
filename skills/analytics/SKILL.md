---
name: ce-analytics
description: Segment / RudderStack e-commerce analytics for Commerce Engine storefronts using @commercengine/analytics. Maps products, carts, orders and users to Segment spec events, deduplicates browser events against Commerce Engine's server-side copies using the `events` that storefront SDK 0.16 mutations return, identifies users after login, and collects ad-platform click identifiers (Meta, Google, TikTok and more).
license: MIT
metadata:
  author: commercengine
  version: "1.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Commerce Engine Analytics

`@commercengine/analytics` turns Commerce Engine objects into [Segment / RudderStack E-Commerce spec (V2)](https://segment.com/docs/connections/spec/ecommerce/v2/) events. Every mapper is a pure function: a `Product`, `Cart`, `CartItem`, `Order` or `User` goes in, and a Segment HTTP Tracking API envelope (`{ type: "track", event, properties, context }`) comes out. The package has no network calls, no storage and no runtime dependencies.

```bash
npm install @commercengine/analytics
```

This skill covers analytics **0.3.0**. Its peer is `@commercengine/storefront-sdk` >= 0.16.0, which `@commercengine/storefront` 0.2.0 and later already include.

## Why It Exists: Two Copies of One Event

Commerce Engine already sends its own **server-side copy** of each event a storefront mutation causes (`Product Added`, `Checkout Started`, `Order Created` and so on) to the analytics integrations configured for the store. The pixel, tag or browser SDK on your storefront sends a **second copy**. Ad platforms such as Meta count the two as one only when both carry the same `event_id` for the same event name.

The package's main job is to build the browser copy **with the id the server used**. Storefront SDK 0.16 returns that id on every mutation that has a server copy:

```
const { data, error, events } = await sdk.cart.addDeleteCartItem(...)
         │     │      └─ the response's top-level `events`: one entry per event the call
         │     │         produced, each with a server-issued `event_id` (a ULID).
         │     │         [] on an error, or when the call changed nothing worth reporting.
         │     └─ null on success
         └─ the response's `content`, unchanged
```

| Channel | Sent by | Goes to | Skill |
|---|---|---|---|
| Browser copy | your storefront, built with this package | Segment / RudderStack browser SDK, ad pixels | this one |
| Server copy | Commerce Engine, automatically | the store's analytics integrations | nothing to build |
| Webhooks | Commerce Engine, server to server | your backend endpoint | `webhooks/` |

Webhooks are a different channel, meant for your backend (syncing, notifications). Don't replay them into a CDP as analytics events: Commerce Engine already sends the server copy, so a replayed webhook adds yet another copy of the same action. A webhook's own `event_id` (its `X-CE-Event-ID` header) is your backend's idempotency key. The id to put on a browser copy is always the one in the mutation response's `events`.

## Impact Levels

- **CRITICAL**: double-counted or phantom conversions
- **HIGH**: wrong quantities or values, lost attribution, broken upgrades
- **MEDIUM**: consent, SSR and typing

## Quick Reference

| Task | Call |
|---|---|
| Initialize once | `const a = createAnalytics({ productUrl, brand })` |
| Send through a browser SDK | `analytics.track(...a.trackArgs(event))` |
| Check for a server copy | `findServerEvent(events, "Coupon Applied")` |
| Map with the server's id | `a.toCouponApplied(data.cart, { events })` |
| Order Completed | `a.toOrderCompleted(order)`: its id is the order number |
| Identify after login | `analytics.identify(...a.identifyArgs(a.toIdentify(data)))` |
| Hosted Checkout events | forward `onAnalyticsEvent` with `a.ambientContext()` |
| Override for one event | `a.withContext({ enrich: false })` |
| Per-event `messageId` / `timestamp` | `a.withEventMetadata(event, { messageId, timestamp })` |

## Decision Tree

```
Which event are you sending?
│
├─ Its SDK call returns an `events` entry (server-backed)
│    sdk.cart:  createCart · addDeleteCartItem · deleteCart · deleteUserCart
│               updateCartAddress · applyCoupon · removeCoupon
│               addToWishlist · removeFromWishlist
│    sdk.order: createOrder · cancelOrder
│    └─ check `error` → findServerEvent(events, name)? → map with { events } → send
│       No entry → send nothing: nothing happened, and the server sent nothing.
│
├─ Order Completed (payment settled; no response carries an id)
│    └─ a.toOrderCompleted(order) → event_id = order number, as on the server's copy
│
├─ No server id: Product Viewed / Clicked, product lists, search, Cart Viewed,
│  Checkout Step Viewed / Completed, Order Updated, Payment Info Entered
│    └─ map and send; no event_id
│
├─ Order Refunded → don't send it from the browser (server-only; there is no mapper)
│
└─ Inside Hosted Checkout → your code gets no `events`; forward onAnalyticsEvent
     with properties unchanged: checkout stamps the server's event_id itself
     (references/destinations.md)
```

## Setup

```typescript
// lib/analytics.ts: create once, import everywhere
import { createAnalytics } from "@commercengine/analytics";

export const a = createAnalytics({
  // Mapping hooks shape `properties.url` and `properties.brand`.
  productUrl: ({ slug }) => (slug ? `/products/${slug}` : undefined),
  brand: () => "Acme",
});
```

Every field is optional:

| Field | Purpose |
|---|---|
| `productUrl(ref)` / `brand(ref)` | build `url` / `brand` from `{ product_id, slug, sku }` |
| `enrich` | `false` collects no destination identifiers; `{ tiktok: false }` turns one platform off |
| `cookies` | a `Cookie` header string or a `(name) => value` getter, for server-side enrichment |
| `context` | extra envelope `context`; explicit values beat collected ones |
| `userId` / `anonymousId` | envelope identity, needed only for server SDK and raw HTTP sends |
| `currency` | last-resort fallback; currency normally comes from the entity |

- **`storeId` is gone** in 0.3.0, because no event id needs it. A 0.2.x initializer that passes `storeId` is now a type error, so delete the field. The SDK's own `createStorefront({ storeId })` is unchanged.
- **`timestamp` and `messageId` are rejected** as defaults, because one id reused across events would make a CDP drop them as duplicates. Attach them to a single envelope with `a.withEventMetadata(event, { messageId, timestamp })`.
- **Bound vs standalone.** Bound mappers take `(entity, options)`. Standalone exports take `(entity, ctx, options)`, so pass `{}` as `ctx` when you have none: `toProductAdded(line, {}, { cart, events })`. `findServerEvent` and `userTraits` exist only as standalone exports.

### Sending

```typescript
// Browser SDKs take (name, properties, options):
analytics.track(...a.trackArgs(a.toCartViewed(cart)));        // Segment analytics.js
rudderanalytics.track(...a.trackArgs(a.toCartViewed(cart)));  // RudderStack JS
```

`trackArgs` returns `[event, properties, { context } | undefined]`, and the third element carries the collected click identifiers. Identity and `messageId` are left out on purpose: the browser SDK owns identity, which is why the dedup id travels in `properties.event_id`. Never pass the envelope itself (`analytics.track(event)`), because the SDK would read it as the event name. For server SDKs and raw HTTP, see `references/destinations.md`.

## Key Patterns

### Gate every server-backed event

```typescript
import { findServerEvent } from "@commercengine/analytics";
import { a } from "./lib/analytics";

const { data, error, events } = await sdk.cart.applyCoupon({ id: cartId }, { coupon_code });
if (error) return;

if (findServerEvent(events, "Coupon Applied")) {
  analytics.track(...a.trackArgs(a.toCouponApplied(data.cart, { events })));
  // properties.event_id === the server copy's event_id
}
```

`events` is also the record of what changed:

- a `createCart` that reuses the shopper's active cart has no `Cart Created`;
- a cancel that leaves the order's status unchanged has no `Order Cancelled`;
- removing a coupon when none is applied has no `Coupon Removed`;
- a quantity change that nets to zero has no `Product Added`.

No entry means no server copy, so a browser copy would report a conversion that never happened.

### Product Added / Removed after `addDeleteCartItem`

`addDeleteCartItem` **sets** a line's quantity, so one call either adds or removes units. The entry's name says which, and its `quantity` is the difference. The mapper reports that difference, and computes `value` from it, instead of the line's total. A line set to 0 is gone from the returned cart, so keep the line from before the call:

```typescript
import type { Cart, CartItem } from "@commercengine/storefront";

async function setLineQuantity(cart: Cart, product_id: string, variant_id: string | null, quantity: number) {
  const isLine = (i: CartItem) => i.product_id === product_id && i.variant_id === variant_id;
  const before = cart.cart_items.find(isLine);

  const { data, error, events } = await sdk.cart.addDeleteCartItem(
    { id: cart.id },
    { product_id, variant_id, quantity },
  );
  if (error) return;
  const after = data.cart.cart_items.find(isLine);

  if (findServerEvent(events, "Product Added")) {
    const line = after ?? before;
    if (line) analytics.track(...a.trackArgs(a.toProductAdded(line, { cart: data.cart, events })));
  }
  if (findServerEvent(events, "Product Removed")) {
    const line = before ?? after;
    if (line) analytics.track(...a.trackArgs(a.toProductRemoved(line, { cart: data.cart, events })));
  }
}
```

Always pass `{ cart: data.cart }`. A `CartItem` has no currency of its own, so the cart supplies `cart_id` and `currency`, and the mapper matches the entry on `cart_id`, `product_id` and `variant_id`.

### `createCart({ items })` returns several entries

It returns `Cart Created` only when it actually creates a cart, and one `Product Added` for **each line it writes**, including when it adds to the active cart it reused:

```typescript
const { data, error, events } = await sdk.cart.createCart({ items });
if (error) return;

if (findServerEvent(events, "Cart Created")) {
  analytics.track(...a.trackArgs(a.toCartCreated(data.cart, { events })));
}
for (const entry of events) {
  if (entry.event !== "Product Added") continue;
  const line = data.cart.cart_items.find(
    (i) => i.product_id === entry.product_id && (i.variant_id ?? null) === entry.variant_id,
  );
  if (line) analytics.track(...a.trackArgs(a.toProductAdded(line, { cart: data.cart, events })));
}
```

### Order Completed: the order number is the id

The server sends `Order Completed` from the payment callback, where no storefront response exists to carry an id. Instead, both copies carry the **order number, exactly as the API returns it** (an order completes only once). Send it on the page the shopper lands on after paying, once payment has succeeded:

```typescript
const status = await sdk.order.getPaymentStatus(orderNumber); // takes a plain string
if (status.error || status.data.status !== "success") return;

const { data, error } = await sdk.order.getOrderDetails({ order_number: orderNumber });
if (error) return;
analytics.track(...a.trackArgs(a.toOrderCompleted(data.order)));
// properties.event_id === data.order.order_number, e.g. "ORD-10023"
```

`createOrder` is a different event. Map its result to `Order Created` with `a.toOrderCreated(data.order, { events })`: the order is placed but not necessarily paid.

> **Upgrade together with the server.** Commerce Engine's server copy of `Order Completed` carries the order number as its `event_id` starting with the same release as analytics 0.3.0. A page still on 0.2.x sends `{store}.order_completed.{order}`, so its `Order Completed`, the one event 0.2.x still deduplicated, stops matching. Upgrade `@commercengine/analytics` to 0.3.0 together with storefront SDK 0.16.

### Two events outside the handshake

- **`Order Refunded`** is reported server-side only. There is no browser mapper (the `OrderRefundedProperties` type describes the server's copy), so don't send one.
- **`Payment Info Entered`** is browser-only, because payment details are submitted as part of order creation. There is no server copy and so no id: `a.toPaymentInfoEntered(cart, { paymentMethod: "upi" })`.

When there is no matching entry, a mapper leaves `event_id` out. It never invents one: a placeholder would be shared by every event that hits the same gap, collapsing them into one conversion. Missing deduplication is better than false deduplication.

## Identify After Login

```typescript
const { data, error } = await sdk.auth.verifyOtp({ otp, otp_token, otp_action: "login" });
if (error) return;
analytics.identify(...a.identifyArgs(a.toIdentify(data)));
```

- `toIdentify` takes an auth result's content as it is (`verifyOtp` or `loginWithPassword`, both `{ user, access_token, refresh_token }`), a `User`, or the JWT `UserInfo` from `sdk.getUserInfo()`, which returns `null` when there is no token, so check it first. Tokens never become traits.
- Identity follows `is_anonymous` only, not login state. A known user gets `userId`, plus `anonymousId` so earlier anonymous activity links to them. An anonymous user gets only `anonymousId`; their id is never used as `userId`.
- `userTraits(data)` (standalone only) returns just the traits: Segment's standard traits plus `customer_id`, `customer_group_id`, `customer_group`, `country_code`, `email_verified` and `phone_verified`.
- `email` is lowercased and trimmed. `phone` becomes digits including the country code (`919897622832`), and is **dropped** when no country code is known. Ad platforms hash these values to match users, and the server applies the same rule, so both sides hash a user identically.

## Common Pitfalls

| Level | Issue | Solution |
|---|---|---|
| CRITICAL | Mapping a mutation without its `events` | The event is built with no `event_id`, so ad platforms count the browser and server copies twice. Destructure `events` from the same call and pass `{ events }`. |
| CRITICAL | Sending a server-backed event without checking for its entry | Gate on `findServerEvent(events, name)`. A reused cart, a cancel that changes nothing, removing a coupon that isn't applied or a quantity change that nets to zero has no entry, and sending anyway reports a conversion that never happened. |
| CRITICAL | Relying on `messageId` for deduplication | `trackArgs` drops `messageId`, and the browser and server copies never share one. The id travels in `properties.event_id`. Make sure each destination reads it: Segment's Facebook Conversions API destination defaults to `messageId` (`references/destinations.md`). |
| CRITICAL | Deriving or minting ids yourself (`{store}.product_added.…`, a UUID, the cart id) | There are only two valid sources: the entry's `event_id`, and the order number for `Order Completed`. Leave `eventId` unset unless you obtained the server's id some other way. |
| HIGH | 0.2.x initializer: `createAnalytics({ storeId })` | Remove `storeId` (a type error in 0.3.0). Upgrade analytics and the storefront SDK together, because a 0.2.x page's `Order Completed` no longer matches the server's id. |
| HIGH | `Product Removed` mapped from the returned cart | A line set to 0 isn't in `data.cart` any more. Map the line as it was before the call. |
| HIGH | Line's total quantity reported as the units added | Pass `events`: the entry's `quantity` is the difference, and `value` follows it. Without an entry the mapper falls back to the line's total. |
| HIGH | `Order Completed` sent after `createOrder` | `createOrder` maps to `Order Created`. `Order Completed` follows a successful payment. |
| HIGH | `Order Refunded` sent from the browser | It is server-only, and there is no mapper. |
| HIGH | `analytics.track(event)`, or keeping only two elements of `trackArgs` | Spread all three: `analytics.track(...a.trackArgs(event))`. The third element carries the click identifiers. |
| HIGH | Standalone mapper called with the bound signature | `toProductAdded(line, { cart, events })` treats the options as `ctx`. Standalone is `(entity, ctx, options)`. Prefer the bound `a.toProductAdded(line, { cart, events })`. |
| MEDIUM | Click ids collected before consent | Collection is unconditional. Under GDPR, CCPA or similar, use `enrich: false` until consent is granted (`a.withContext({ enrich: false })`). |
| MEDIUM | `a.trackEvent("Cart Updated", a.cartProperties(cart))` fails to type-check | A custom name needs an open object: spread the bag (`{ ...a.cartProperties(cart) }`), or declare the event on `EcommerceEventProperties` (`references/mappers.md`). |
| MEDIUM | No identifiers on events built during SSR | Server-side, collection reads nothing unless you pass `cookies` (a `Cookie` header or a `CookieAdapter`'s `get`). |

## References

| Reference | When to Use |
|---|---|
| `references/mappers.md` | Every mapper: its event, input, the SDK call it follows and its options; the `events` entry fields; property builders and custom events |
| `references/destinations.md` | Browser, server and HTTP dispatch; Hosted Checkout forwarding; which field each destination deduplicates on; click identifiers, consent and SSR cookies |

## See Also

- `cart-checkout/`: the cart mutations whose results carry `events`, and Hosted Checkout setup
- `orders/`: `createOrder`, `getPaymentStatus` and `cancelOrder`
- `auth/`: `verifyOtp` and `loginWithPassword`, whose results feed `toIdentify`
- `setup/`: SDK install, `createStorefront`, and public vs session clients
- `webhooks/`: server-to-server events sent to your backend, a separate channel from CDP analytics events

## Documentation

- **Package**: https://www.npmjs.com/package/@commercengine/analytics
- **Segment E-Commerce spec (V2)**: https://segment.com/docs/connections/spec/ecommerce/v2/
- **Response `events` entry**: https://llm-docs.commercengine.io/storefront/schemas/StorefrontResponseEvent
- **LLM Reference (cart items)**: https://llm-docs.commercengine.io/storefront/operations/update-cart
- **LLM Reference (create cart)**: https://llm-docs.commercengine.io/storefront/operations/create-cart
