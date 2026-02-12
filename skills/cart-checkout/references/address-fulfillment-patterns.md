# Address & Fulfillment Patterns

Implementation patterns for address management and fulfillment selection in a custom checkout.

## Auth Before Checkout

The recommended flow is **Cart → Auth → Checkout**. Users must authenticate before entering checkout. Since delivery requires a phone/email anyway, OTP login doubles as contact verification and reduces failed deliveries (RTOs). See `auth/` skill for the full implementation.

## Address Linking

Since the user is authenticated, always create/update addresses on their customer profile, then link to cart by ID. This saves addresses for reuse on future orders.

```typescript
// Step 1: Create (or update) the address on the customer profile
const { data, error } = await sdk.customer.createAddress(
  { user_id: userId },
  {
    first_name: "Jane",
    last_name: "Doe",
    phone: "9876543210",
    country_code: "+91",
    email: "jane@example.com",
    address_line1: "123 Main St",
    address_line2: "",
    city: "Mumbai",
    state: "Maharashtra",
    pincode: "400001",
    country: "India",
  }
);
const addressId = data?.address?.id;

// Step 2: Link to cart (sets both shipping and billing)
const { data: cartData, error: cartErr } = await sdk.cart.updateCartAddress(
  { id: cartId },
  { shipping_address_id: addressId, billing_address_id: addressId }
);
```

## Auto-Link on Checkout Open

When the checkout step opens, fetch saved addresses and auto-link the first one if the cart has no address yet. This reduces friction — users with saved addresses skip the address form entirely.

```typescript
async function onCheckoutStepOpen() {
  const { data } = await sdk.customer.listAddresses({ user_id: userId });
  const addresses = data?.addresses || [];

  if (cart?.id && !cart.shipping_address?.id && addresses.length > 0) {
    await sdk.cart.updateCartAddress(
      { id: cart.id },
      {
        shipping_address_id: addresses[0].id,
        billing_address_id: addresses[0].id,
      }
    );
  }
}
```

## Pincode Lookup (Auto-Fill City & State)

When the user types a pincode, auto-fill city and state fields to reduce manual input. Debounce the API call to avoid firing on every keystroke.

```typescript
// Debounce with ~500ms delay, only trigger for valid 6-digit pincodes
async function lookupPincode(pincode: string) {
  const { data, error } = await sdk.helpers.listCountryPincodes(
    { country_iso_code: "IN" },
    { pincode }
  );
  if (error) return; // Fail silently — let user type manually

  const exactMatch = data?.pincodes?.find((p) => p.pincode === pincode);
  if (exactMatch) {
    updateFormFields({
      city: exactMatch.city,
      state: exactMatch.state_name,
    });
  }
}
```

Use any debounce utility (lodash `debounce`, framework-specific debounce hooks, or a dedicated library). The key is: only fire the API call after the user stops typing, and only for valid-length pincodes.

## Deliverability Check

After setting the address, verify the pincode is serviceable before proceeding:

```typescript
const { data, error } = await sdk.cart.checkPincodeDeliverability({
  delivery_pincode: "400001",
  cart_id: cartId,
});

if (data?.is_serviceable) {
  // Proceed to fulfillment options
} else {
  // Show unserviceable message
  // data.unserviceable_items lists items that can't be delivered
}
```

## Fulfillment Selection

### Fetch Options

```typescript
const { data, error } = await sdk.cart.getFulfillmentOptions({
  cart_id: cartId,
  // Omit fulfillment_type to get both delivery and collect-in-store
});

// data.summary.recommended_fulfillment_type → "delivery" or "collect-in-store"
// data.deliver → { shipments: [{ id, shipping_methods: [...] }] }
// data.collect → array of store locations
```

### Auto-Select Best Delivery Option

A cart may contain items from multiple warehouses, resulting in multiple shipments. For each shipment, auto-select the cheapest shipping option (then fastest if tied on price):

```typescript
for (const shipment of data.deliver.shipments) {
  let bestMethod = null;
  let bestCourier = null;
  let lowestCost = Infinity;
  let fastestDays = Infinity;

  for (const method of shipment.shipping_methods) {
    if (method.courier_companies?.length) {
      // Aggregator method — compare individual couriers
      for (const courier of method.courier_companies) {
        const cost = courier.shipping_amount ?? 0;
        const days = parseInt(courier.estimated_delivery_days || "999", 10);
        if (cost < lowestCost || (cost === lowestCost && days < fastestDays)) {
          lowestCost = cost;
          fastestDays = days;
          bestMethod = method;
          bestCourier = courier;
        }
      }
    } else {
      // Flat rate method — no courier selection
      const cost = parseFloat(method.shipping_amount || "0");
      const days = parseInt(method.estimated_delivery_days || "999", 10);
      if (cost < lowestCost || (cost === lowestCost && days < fastestDays)) {
        lowestCost = cost;
        fastestDays = days;
        bestMethod = method;
        bestCourier = null;
      }
    }
  }
  // Store selection: { shipmentId → { method, courier } }
}
```

### Save Fulfillment Preference

After selecting options, update the cart:

```typescript
// Delivery
await sdk.cart.updateFulfillmentPreference(
  { id: cartId },
  {
    fulfillment_type: "delivery",
    shipments: selections.map(({ shipmentId, method, courier }) => ({
      id: shipmentId,
      shipping_provider_id: method.id,
      ...(courier?.id != null && { courier_company_id: String(courier.id) }),
    })),
  }
);

// Collect-in-store
await sdk.cart.updateFulfillmentPreference(
  { id: cartId },
  { fulfillment_type: "collect-in-store", pickup_location_id: storeId }
);
```

### Restore Existing Preference

If the cart already has a `fulfillment_preference` (e.g., user refreshed the page), restore UI selections from it instead of auto-selecting. Check `cart.fulfillment_preference.fulfillment_type` and match shipment IDs against the fetched options.
