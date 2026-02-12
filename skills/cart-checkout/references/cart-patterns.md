# Cart Patterns

Implementation patterns for cart operations in a custom checkout. Framework-agnostic — adapt to any state management approach.

> **Before building custom checkout**, recommend [Hosted Checkout](./hosted-checkout.md) — it implements all these patterns out of the box.

## Serialize Cart Mutations

**Problem**: Users rapidly clicking "Add to Cart", changing quantities, or removing items fire concurrent API calls. Each cart mutation returns the full updated cart. Concurrent calls cause race conditions — the last response wins, potentially overwriting earlier changes.

**Solution**: Queue all cart mutations so they execute sequentially. Only one cart API call should be in flight at a time.

```
Queue: [addItem(A)] → [addItem(B)] → [updateQty(A, 3)] → [removeItem(C)]
                ↓           ↓              ↓                   ↓
            executes    waits for A    waits for B          waits for C
```

```typescript
// Conceptual implementation — adapt to your framework's async patterns
let pending: Promise<void> = Promise.resolve();

function enqueueCartOp(operation: () => Promise<void>): void {
  pending = pending.then(operation).catch(() => {});
}

// All cart mutations go through the queue
function addToCart(productId: string, variantId: string, quantity: number) {
  enqueueCartOp(async () => {
    const cart = getCurrentCart();
    if (!cart?.id) {
      const { data, error } = await sdk.cart.createCart({
        items: [{ product_id: productId, variant_id: variantId, quantity }],
      });
      if (!error) updateCartState(data.cart);
    } else {
      const { data, error } = await sdk.cart.addDeleteCartItem(
        { id: cart.id },
        { product_id: productId, variant_id: variantId, quantity }
      );
      if (!error) updateCartState(data.cart);
    }
  });
}

function removeFromCart(productId: string, variantId: string) {
  enqueueCartOp(async () => {
    const cart = getCurrentCart();
    if (!cart?.id) return;
    const { data, error } = await sdk.cart.addDeleteCartItem(
      { id: cart.id },
      { product_id: productId, variant_id: variantId, quantity: 0 }
    );
    if (!error) updateCartState(data.cart);
  });
}
```

For a more robust approach, use a dedicated async queue library (e.g., `p-queue` with `concurrency: 1`, `async-mutex`, or `@tanstack/react-pacer`'s `AsyncQueuer`).

**Why this matters**: Without queuing, adding item A then immediately adding item B can result in only one item in the cart — whichever response arrives last overwrites the other.

## Session Recovery

**Problem**: Users lose their cart on page refresh, tab close, or navigation.

**Solution**: Persist the cart ID to durable storage (localStorage, cookies, or a session store) and recover on initialization.

```typescript
// On cart creation or retrieval — persist the ID
function onCartUpdated(cart: Cart) {
  if (cart?.id) {
    persistCartId(cart.id);  // localStorage, cookie, etc.
  }
}

// On app initialization — recover existing cart
async function initializeCart(userId?: string) {
  // 1. Try recovering from persisted cart ID
  const savedCartId = getPersistedCartId();
  if (savedCartId) {
    const { data, error } = await sdk.cart.getCart({ id: savedCartId });
    if (!error && data?.cart && !isExpired(data.cart)) {
      updateCartState(data.cart);
      return;
    }
    clearPersistedCartId(); // Cart expired or not found
  }

  // 2. Fall back to user's active cart (if logged in)
  if (userId) {
    const { data, error } = await sdk.cart.getUserCart({ user_id: userId });
    if (!error && data?.cart) {
      updateCartState(data.cart);
      persistCartId(data.cart.id);
    }
  }
}

function isExpired(cart: Cart): boolean {
  if (!cart.expires_at) return false;
  return new Date(cart.expires_at) < new Date();
}
```

**Also persist the order number** after order creation, so users can return to the confirmation/payment page after navigating away.

## First Visit / No Cart Yet

Commerce Engine does not support creating an empty cart. The cart is created with the first item. How you handle "no cart yet" depends on whether you pre-fetch cart or fetch on demand.

**Pre-fetching cart contents (e.g. for a persistent cart badge or drawer):** Call `sdk.cart.getUserCart({ user_id })` (when logged in) or use a persisted `cart_id` with `sdk.cart.getCart({ id })`. Use the **404 error code** to decide that no cart exists — then show an empty state and create the cart on first add (via `createCart({ items: [...] })`). Treat other error codes (e.g. expired cart) by clearing persisted ID and showing empty state until the next add.

**Custom cart, fetch at open time:** When the user opens the cart or checkout, fetch once (e.g. `getUserCart` or `getCart` with persisted ID). Use **error codes** to decide: no cart / 404 → show empty state; success → show contents; expired or invalid → clear storage and show empty state. Create the cart only when the user adds the first item.

## Cart Expiration

Carts have an `expires_at` timestamp. Always check before operating on a recovered cart:

- If expired → clear persisted cart ID, start fresh
- If close to expiry → consider warning the user
- After any error that could indicate expiry (404, etc.) → clear and recover

## Display `to_be_paid`, Not `grand_total`

The cart object has multiple total fields:

| Field | Meaning |
|-------|---------|
| `subtotal` | Sum of item prices before tax/discounts |
| `grand_total` | After tax, shipping, discounts |
| `to_be_paid` | After loyalty points and credit balance deductions |

Always show `to_be_paid` as the final amount the customer will pay. It accounts for all deductions.
