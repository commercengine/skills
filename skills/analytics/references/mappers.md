# Mapper Reference

Every mapper is exported in two forms:

- **Bound** (recommended): `a.toX(input, options?)` on the client `createAnalytics()` returns. The initializer's context (hooks, `enrich`, `cookies`, identity) applies automatically.
- **Standalone**: `toX(input, ctx?, options?)`. `ctx` is a `MapContext` and comes second, so pass `{}` when you have none: `toProductAdded(line, {}, { cart, events })`.

`toProductsSearched(query)`, `toCartViewed(cart)` and `toIdentify(source)` take no options. `findServerEvent`, `withEventId`, `userTraits` and `normalizePhone` / `normalizeEmail` are standalone only.

Every mapper returns a `TrackEvent` envelope. Send it with `analytics.track(...a.trackArgs(event))`.

## Server-Backed Events

Pass the `events` of the **same** response, and send only when `findServerEvent(events, name)` finds the entry. Each mapper picks its own entry out of `events` and copies its `event_id`. An explicit, non-blank `eventId` option wins over the entry. Leave it unset unless you got the server's id some other way.

| Event | Bound mapper | Input | After this SDK call | Entry matched on | Options |
|---|---|---|---|---|---|
| `Cart Created` | `a.toCartCreated` | `data.cart` | `sdk.cart.createCart` | `cart_id` | `events`, `eventId` |
| `Cart Deleted` | `a.toCartDeleted` | the `Cart` **as it was before** the call (deleting returns no cart) | `sdk.cart.deleteCart` / `deleteUserCart` | `cart_id` | `events`, `eventId` |
| `Product Added` | `a.toProductAdded` | the `CartItem` after the call (or before it, if gone) | `sdk.cart.addDeleteCartItem`, `sdk.cart.createCart` | `cart_id`, `product_id`, `variant_id` | `cart`, `cartId`, `events`, `eventId` |
| `Product Removed` | `a.toProductRemoved` | the `CartItem` **before** the call | `sdk.cart.addDeleteCartItem` | `cart_id`, `product_id`, `variant_id` | `cart`, `cartId`, `events`, `eventId` |
| `Checkout Started` | `a.toCheckoutStarted` | `data.cart` | `sdk.cart.updateCartAddress` | `cart_id` | `orderId`, `affiliation`, `events`, `eventId` |
| `Coupon Applied` | `a.toCouponApplied` | `data.cart` | `sdk.cart.applyCoupon` | `cart_id` | `couponCode`, `events`, `eventId` |
| `Coupon Removed` | `a.toCouponRemoved` | `data.cart` | `sdk.cart.removeCoupon` | `cart_id` | `couponCode`, `events`, `eventId` |
| `Product Added to Wishlist` | `a.toProductAddedToWishlist` | a wishlist `Item` from `data.products`, or the catalog `Product` / `ProductDetail` on screen | `sdk.cart.addToWishlist` | `product_id`, `variant_id` | `variant`, `events`, `eventId` |
| `Product Removed from Wishlist` | `a.toProductRemovedFromWishlist` | the wishlist `Item` being removed, or the `Product` | `sdk.cart.removeFromWishlist` | `product_id`, `variant_id` | `variant`, `events`, `eventId` |
| `Order Created` | `a.toOrderCreated` | `data.order` | `sdk.order.createOrder` | `order_number` | `affiliation`, `events`, `eventId` |
| `Order Cancelled` | `a.toOrderCancelled` | `data.order` (an `OrderDetail`, optional in the type, so check it) | `sdk.order.cancelOrder` | `order_number` | `affiliation`, `events`, `eventId` |

Notes per event:

- **`Product Added` / `Product Removed`.** Pass `cart` (preferred) or `cartId`. With neither, `cart_id` isn't compared. The entry's `quantity` (the units this call added or removed) replaces the line's own quantity. `value` is `price × quantity`, rounded to 4 decimal places, and is left out when it isn't positive. A zero or missing quantity counts as 1 for `value`. `variant_id` is compared as "no variant" when both sides are `null`, `undefined` or `""`.
- **`Checkout Started`.** The server reports it **each time an address is set on the cart**, even an unchanged one, so send the browser copy from the same `updateCartAddress` result.
- **`Coupon Removed`.** The cart `removeCoupon` returns no longer carries the code, so the mapper reads it from the entry (or from `couponCode`). `coupon_id` and `discount` are left out. If an `applyCoupon` response also carries a `Coupon Removed` entry (for a code the new one replaced), map it with `a.toCouponRemoved(data.cart, { events })` as well. It reads the old code from its own entry.
- **Wishlist.** The request names a product and a variant (`variant_id: null` only for a product without variants), and the entry repeats both. An `Item` is matched on its own product and variant. A `Product` is matched on its id and then described as the variant the entry names. Pass `variant` only to override that.

```typescript
// Wishlist: map the Item the response returns.
const { data, error, events } = await sdk.cart.addToWishlist({ product_id, variant_id });
if (error) return;
if (findServerEvent(events, "Product Added to Wishlist")) {
  const item = data.products.find(
    (p) => p.product_id === product_id && (p.variant_id ?? null) === variant_id,
  );
  if (item) analytics.track(...a.trackArgs(a.toProductAddedToWishlist(item, { events })));
}

// Delete: the response has no cart, so map the one you already hold.
const deleted = await sdk.cart.deleteCart({ id: cart.id });
if (!deleted.error && findServerEvent(deleted.events, "Cart Deleted")) {
  analytics.track(...a.trackArgs(a.toCartDeleted(cart, { events: deleted.events })));
}

// Cancel: data.order is optional in the type.
const cancelled = await sdk.order.cancelOrder(
  { order_number },
  { cancellation_reason: "Changed my mind", refund_mode: "original-payment-mode" },
);
if (!cancelled.error && cancelled.data.order && findServerEvent(cancelled.events, "Order Cancelled")) {
  analytics.track(...a.trackArgs(a.toOrderCancelled(cancelled.data.order, { events: cancelled.events })));
}
```

## Order Completed

| Event | Bound mapper | Input | When | Id | Options |
|---|---|---|---|---|---|
| `Order Completed` | `a.toOrderCompleted` | `Order` / `OrderDetail` (`getOrderDetails` → `data.order`) | payment has succeeded (`getPaymentStatus` → `status: "success"`) | `order_number`, verbatim | `affiliation`, `eventId` |

It takes no `events` option: its id never comes from a response.

## Events With No Server Id

These are built without `event_id`, because no response returns an entry for them.

| Event | Bound mapper | Input | Typical source | Options |
|---|---|---|---|---|
| `Products Searched` | `a.toProductsSearched` | the query string | your search box | none |
| `Product Viewed` | `a.toProductViewed` | `Product`, `ProductDetail` or `Item` | `sdk.catalog.getProductDetail({ product_id })` → `data.product` | `variant` |
| `Product Clicked` | `a.toProductClicked` | `Product`, `ProductDetail` or `Item` | a product card click | `position` (1-based), `variant` |
| `Product List Viewed` | `a.toProductListViewed` | an array of any of those | `listProducts` → `data.products`, `searchProducts` → `data.skus` | `listId`, `category` |
| `Product List Filtered` | `a.toProductListFiltered` | an array of any of those | a filtered `listProducts` | `listId`, `category`, `filters`, `sorts` |
| `Cart Viewed` | `a.toCartViewed` | `Cart` | `sdk.cart.getCart({ id })` / `getUserCart()` → `data.cart` | none |
| `Checkout Step Viewed` / `Completed` | `a.toCheckoutStepViewed` / `a.toCheckoutStepCompleted` | `Cart` | your checkout steps | `step`, `shippingMethod`, `paymentMethod` |
| `Payment Info Entered` | `a.toPaymentInfoEntered` | `Cart` | the shopper chooses a payment method (browser-only) | `step`, `shippingMethod`, `paymentMethod`, `orderId`, `eventId` |
| `Order Updated` | `a.toOrderUpdated` | `Order` / `OrderDetail` | your own order updates | `affiliation` |

List mappers set each product's `position` from its index (1-based). `filters` and `sorts` are `{ type, value }` objects.

## Products and Variants

`Product` and `ProductDetail` describe the parent product: its variants live in `variants[]`, and in SDK 0.16 its own `pricing` is `null` when it has variants. Pass the selected variant as an object or an id: `a.toProductViewed(data.product, { variant: selectedVariantId })`. Without one, the product's default variant (`is_default`) is used, in list events too. A resolved variant overrides `variant` / `variant_id` / `variant_slug`, and also `sku`, `price`, `image_url` and the stock and promotion flags. `currency` comes from the variant whose price is reported, then the product's pricing, then `ctx.currency`.

`Item`, `CartItem` and `OrderItem` already describe one variant inline. An `Item`'s `slug` is the variant's when `variant_id` is set.

## What Each Event Carries

- **Product object** (every product event and every entry in `products`): `product_id`, `sku`, `name`, `variant` (variant name), `variant_id`, `variant_slug`, `variant_name`, `product_slug`, `product_type`, `brand` (from your hook), `category`, `category_ids`, `category_slugs`, `tags`, stock / backorder / subscription / promotion flags, `price` (selling price), `listing_price`, `selling_price`, `selling_price_excluding_tax`, tax fields, line discounts, subscription fields, compact `attributes` keyed by attribute key or name, `seller_id`, `quantity`, `position`, `url` (from your hook) and `image_url`. Empty slugs are left out.
- **Cart bag** (`Cart Viewed`, `Cart Created`, `Cart Deleted`, and inside `Checkout Started`): `cart_id`, `currency`, `value` / `total` (grand total), `subtotal`, `subtotal_including_tax`, tax, `shipping`, `discount` (`total_savings`, else coupon + promotion), coupon and promotion amounts and savings, loyalty and credit amounts, `cart_items_count`, `status` and `products`.
- **`Checkout Started`** adds `value` (grand total), `revenue` (subtotal), `shipping`, `tax`, `discount`, `coupon`, plus `order_id` and `affiliation` when passed.
- **Order bag** (every order event): `order_id` (the order number), `cart_id`, `checkout_id` (the same cart id, since Commerce Engine has no separate checkout id), `value` / `total` (grand total), `revenue` / `subtotal`, `shipping`, `tax`, `discount`, coupon and promotion fields, `status`, `payment_status`, `order_date`, loyalty and credit amounts, `currency` and `products`. `Order Completed` adds `subtotal`.
- **Checkout step / `Payment Info Entered`**: `checkout_id` and `cart_id` (both the cart id), `step`, `payment_method`, and `shipping_method`, derived from `cart.fulfillment_preference` (delivery provider / courier, collect-in-store pickup location, or both). `Payment Info Entered` adds `order_id` when passed.
- **`Coupon Applied` / `Removed`**: `cart_id`, `coupon_id` (Applied only), `coupon_name` and `coupon_code` (both the code), `discount` (Applied only) and `currency`.

Addresses, shipments, raw promotion objects, inventory and seller details are left out by default. Add them yourself only when a destination needs them. Non-finite numbers (`Infinity`, `NaN`) are stripped from every bag.

## The `events` Entries

`events` is `StorefrontResponseEvent[]`. Every entry has `event` (the discriminator) and `event_id` (a server-issued ULID), plus:

| `event` | Other fields |
|---|---|
| `Cart Created`, `Cart Deleted`, `Checkout Started` | `cart_id` |
| `Product Added`, `Product Removed` | `cart_id`, `product_id`, `variant_id` (`null` when the product has no variants), `quantity` (units added or removed) |
| `Coupon Applied`, `Coupon Removed` | `cart_id`, `coupon_code` |
| `Order Created` | `order_number`, `cart_id` (`null` when the order didn't come from a cart) |
| `Order Cancelled` | `order_number` |
| `Product Added to Wishlist`, `Product Removed from Wishlist` | `product_id`, `variant_id` |

The methods that return `events` are listed in the Decision Tree in `SKILL.md`. Every other SDK method keeps the plain `ApiResult<T>`, without `events` in its type. Code that builds these results itself (test doubles, wrappers) must now include `events` (for example `events: []`).

## `findServerEvent(events, name, match?)`

It returns the first entry with that name whose fields equal every defined key of `match`, typed for that event, or `undefined`. Keys left `undefined` aren't compared. It accepts `undefined` and skips malformed entries, so it never throws.

```typescript
const entry = findServerEvent(events, "Product Added", { cart_id: cart.id, product_id, variant_id: null });
entry?.quantity; // typed: number
```

## Custom Events and `trackEvent`

`a.trackEvent(name, properties)` builds any event. Spec names are type-checked against their properties; a custom name accepts an open object. Use it for spec events with no source entity: `Promotion Viewed` / `Clicked`, `Coupon Entered` / `Denied`, `Product Shared`, `Cart Shared`, `Product Reviewed` and `Wishlist Product Added to Cart`.

The bound property builders inherit your hooks: `a.productProperties(source, { variant, position })`, `a.cartItemProperties(item)`, `a.orderItemProperties(item)`, `a.cartProperties(cart)`, `a.checkoutStepProperties(cart, options)`, `a.paymentInfoEnteredProperties(cart, options)` and `a.orderProperties(order, { affiliation })`.

```typescript
import { findServerEvent, withEventId } from "@commercengine/analytics";

// A custom name needs an open object, so spread the bag:
analytics.track(...a.trackArgs(a.trackEvent("Cart Updated", { ...a.cartProperties(data.cart) })));

// A spec event with no source entity:
analytics.track(...a.trackArgs(a.trackEvent("Promotion Viewed", { promotion_id: "promo_1", name: "Sale" })));

// An event built with trackEvent carries no event_id. For one the server also sends, attach it:
const entry = findServerEvent(events, "Cart Created");
if (entry) {
  const properties = withEventId(a.cartProperties(data.cart), undefined, entry.event_id);
  analytics.track(...a.trackArgs(a.trackEvent("Cart Created", properties)));
}
```

`withEventId(properties, explicitId?, knownId?)` adds `event_id`, preferring a non-blank `explicitId` over `knownId`, and returns the properties unchanged when neither applies. It used to be `withEventId(eventName, properties, ctx, explicitId?)` in 0.2.x.

To type custom events strictly, augment the spec map:

```typescript
import type { CartProperties, OrderProperties } from "@commercengine/analytics";

declare module "@commercengine/analytics" {
  interface EcommerceEventProperties {
    "Cart Updated": CartProperties;
    "Payment Successful": OrderProperties & { payment_method?: string };
  }
}
```

`Cart Created`, `Cart Deleted` and `Order Created` are typed entries of `EcommerceEventProperties` in 0.3.0. Remove any declaration merge from 0.2.x that typed them differently.

## Removed in 0.3.0

- `toOrderRefunded` and `OrderRefundedOptions`: refunds are server-only. The `OrderRefundedProperties` type remains.
- The id derivation API: `deriveEventId`, `buildEventId`, `eventSlug`, `normalizePart`, `EVENT_ID_KEY_SPEC`, `EVENT_ID_OPTIONAL_KEYS` and `EVENT_ID_ABSENT_PART`.
- `storeId` on `MapContext` and `createAnalytics`.
