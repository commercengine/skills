# Sending, Deduplication Fields and Destination Identifiers

## Sending an Envelope

A mapper returns the Segment / RudderStack **HTTP Tracking API** payload, not the argument list of a browser SDK. Dispatch it the way your target expects.

### Browser SDKs (Segment analytics.js, RudderStack JS)

```typescript
const event = a.toCartViewed(cart);
analytics.track(...a.trackArgs(event));        // Segment
rudderanalytics.track(...a.trackArgs(event));  // RudderStack

const { data, error } = await sdk.auth.verifyOtp({ otp, otp_token, otp_action: "login" });
if (!error) analytics.identify(...a.identifyArgs(a.toIdentify(data)));
```

`trackArgs(event)` returns `[event.event, event.properties, { context } | undefined]`. Identity is ambient: the SDK attaches the `userId` / `anonymousId` it holds from `identify()`. `trackArgs` leaves out the envelope's identity, `timestamp` and `messageId`, which is why the dedup id lives in `properties.event_id`. `identifyArgs(event)` returns `[userId, traits]`.

### Server SDKs and raw HTTP

A server has no ambient session and no `document.cookie`. Pass identity, and the request's cookies, through the initializer so they land on the envelope:

```typescript
import { createAnalytics, findServerEvent } from "@commercengine/analytics";

const server = createAnalytics({
  userId, // server SDKs need a userId or an anonymousId
  cookies: request.headers.get("cookie") ?? undefined,
});

const { data, error, events } = await sdk.cart.applyCoupon({ id: cartId }, { coupon_code });
if (error) return;
if (findServerEvent(events, "Coupon Applied")) {
  const event = server.toCouponApplied(data.cart, { events });

  // Server SDK object form (@segment/analytics-node, @rudderstack/rudder-sdk-node):
  nodeAnalytics.track({
    userId: event.userId,
    event: event.event,
    properties: event.properties,
    context: event.context,
  });

  // Or raw HTTP ingestion: POST the envelope unchanged, with your CDP's auth headers.
  await fetch("https://cdp.example.com/v1/track", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(event),
  });
}
```

`cookies` also accepts a getter, and a Commerce Engine `CookieAdapter`'s `get` fits: `createAnalytics({ cookies: (name) => adapter.get(name) })`.

To set `messageId` or `timestamp` on one envelope (never as defaults), use `a.withEventMetadata(event, { messageId: crypto.randomUUID(), timestamp: new Date().toISOString() })`.

### Hosted Checkout

Cart and checkout calls made inside Hosted Checkout don't return `events` to your code. Instead, the checkout emits its own analytics envelopes, `track` and `identify`, which the parent page forwards. `initCheckout` takes `onAnalyticsEvent`, and the vanilla checkout also emits `analytics:event`. The checkout iframe is cross-origin and can't read your first-party cookies, so attach the identifiers while forwarding:

```typescript
import { initCheckout } from "@commercengine/checkout";
import { a } from "./lib/analytics";

initCheckout({
  storeId: import.meta.env.VITE_STORE_ID,
  apiKey: import.meta.env.VITE_API_KEY,
  authMode: "provided", // plus the two-way token sync from cart-checkout/
  onAnalyticsEvent: (event) => {
    if (event.type === "track") {
      analytics.track(event.event, event.properties, a.ambientContext());
    } else {
      analytics.identify(event.userId, event.traits);
    }
  },
});
```

Use the bound `a.ambientContext()`: it applies the initializer's `enrich` and `cookies`, so a consent opt-out covers forwarded events too. The standalone `ambientContext()` knows neither. Both return `undefined` when there is nothing to attach.

Forward `event.properties` unchanged. Checkout builds its envelopes with the same mappers, and stamps each server-backed event with the `event_id` from its own mutation's `events`, so the forwarded copy already deduplicates against the server's. Don't derive, overwrite or strip `event_id` in the parent page. An envelope without one is an event with no server copy to match, or one sent before the server reported an id.

## Which Field Each Destination Deduplicates On

The browser copy and the server copy share `properties.event_id`, and both use `properties.order_id` for the order number. Their `messageId`s always differ. Check that each destination reads the shared field:

| Destination | Meta event ID taken from | What to do |
|---|---|---|
| RudderStack Facebook Pixel and Conversions | `properties.event_id`, falling back to `messageId` | nothing |
| Segment Facebook Conversions API (Actions) | its Event ID field, which defaults to `messageId` | map Event ID to `properties.event_id` |
| Segment Facebook Pixel (classic, device mode) | always `messageId` | it can't carry the server's id; deduplication needs a destination that reads `properties.event_id` |

GA4 doesn't use an event id for purchases: it matches them on `transaction_id`, which Segment and RudderStack fill from `properties.order_id`. The order mappers set `order_id` to the order number, so it already matches the server's copy.

## Destination Identifiers, Collected Automatically

Ad platforms match a conversion back to the click that caused it with an identifier their own tag wrote into a first-party cookie, or a click id appended to the landing URL. Every **track** mapper collects whatever is available and puts it at the **top level of `context`**, which is where destinations read it. `trackArgs` delivers it as the third argument. `identify` envelopes aren't enriched.

| Platform | `context` keys | Read from |
|---|---|---|
| Meta | `fbc`, `fbp` | `_fbc`, `_fbp` cookies |
| Google | `gclid`, `gbraid`, `wbraid`, `ga_client_id` | URL params; `_gcl_aw` and `_ga` cookies |
| TikTok | `ttclid`, `ttp` | URL param; `_ttp` cookie |
| Snapchat | `sccid`, `scid` | `ScCid` (or `sc_click_id`) URL param; `_scid` cookie |
| Reddit | `rdt_cid`, `rdt_uuid` | URL param; `_rdt_uuid` cookie |
| Microsoft | `msclkid` | URL param |
| LinkedIn | `li_fat_id` | URL param |
| X / Twitter | `twclid` | URL param |
| Pinterest | `epik` | URL param, else `_epik` cookie |

- Anything absent is simply not collected: no placeholders, no empty strings. A new platform is a package change, not an app change.
- **Verification status.** Meta's `context.fbc` / `context.fbp` are confirmed against RudderStack's Facebook Conversions destination. The others use each platform's conventional key name, so confirm them against your destination's field mapping before relying on them.
- `fbc` is read only from the `_fbc` cookie the Meta pixel writes. The package never builds one from `fbclid`.
- Only Google's default cookie names (`_ga`, `_gcl_aw`) are read. A custom cookie prefix isn't collected.
- URL click ids exist only on the landing page's URL. The package reads them and doesn't persist them; it never writes cookies or storage.
- Collection never throws. A blocked `document.cookie`, a failing cookie getter or one platform erroring leaves the event unenriched rather than suppressing it.

### Opting out and consent

Collection is unconditional: click ids are read from the URL even before any consent dialog resolves. This package has no notion of consent, so enforce it yourself.

```typescript
const noTracking = createAnalytics({ enrich: false });                 // collect nothing
const noTikTok = createAnalytics({ enrich: { tiktok: false } });       // one platform off
const pinnedFbp = createAnalytics({ context: { fbp: "fb.1.123.456" } }); // explicit values win
const once = a.withContext({ enrich: false });                         // one-event override
analytics.track(...once.trackArgs(once.toCartViewed(cart)));
```

Platform names for `enrich` are `meta`, `google`, `tiktok`, `snapchat`, `reddit`, `microsoft`, `linkedin`, `twitter` and `pinterest` (exported as `ENRICHER_NAMES`). `withContext` merges nested `context` objects: an explicit value wins, and `undefined` leaves the initializer's value in place.

### Server-side rendering

During SSR there is no `document.cookie` or `window.location`, so events carry no identifiers unless you pass `cookies` (a `Cookie` header string or a getter). URL click ids are still unavailable there. `a.readAmbientIdentifiers()` returns the collected `context` fragment, and `a.ambientContext()` returns `{ context }`, if you need them for an envelope you build yourself.
