# Payment Patterns

Implementation patterns for payment method discovery, validation, and payment flow in a custom checkout.

## Payment Method Discovery

Fetch available payment methods and group by type for UI display:

```typescript
const { data, error } = await sdk.payments.listPaymentMethods({
  payment_provider_slug: "juspay",
});

const provider = data?.payment_methods?.find(
  (p) => "payment_provider_slug" in p && p.payment_provider_slug === "juspay"
);

// Group provider.details by payment_method_type → ["UPI", "CARD", "NB", "WALLET"]
// Each group contains options with: payment_method, description, supported_reference_ids
```

Recommended display order: UPI, CARD, NB (net banking), WALLET.

For NB and WALLET types, deduplicate options by `juspay_bank_code` — the same bank may appear multiple times with different descriptions.

## Parallel Data Fetching

When the payment step opens, fetch all payment-related data in parallel to minimize wait time:

```typescript
await Promise.all([
  fetchCustomerCards(),       // sdk.customer.listCustomerCards({ customer_id })
  fetchSavedPaymentMethods(), // sdk.customer.listSavedPaymentMethods({ customer_id })
  fetchPaymentMethods(),      // sdk.payments.listPaymentMethods({ payment_provider_slug })
]);
```

## Validating a Payment Method

Before attempting payment, verify the specific method (e.g., VISA, UPI_COLLECT) is supported by a configured gateway:

```typescript
function checkPaymentMethodSupport(
  groupedMethods: Array<{ type: string; options: PaymentMethodOption[] }>,
  paymentMethodType: string, // "CARD", "UPI", "NB", "WALLET"
  paymentMethod: string       // "VISA", "MASTER", "UPI_COLLECT", "NB_HDFC", etc.
): { isSupported: boolean; gatewayReferenceId: string | null } {
  const group = groupedMethods.find(
    (g) => g.type.toUpperCase() === paymentMethodType.toUpperCase()
  );
  if (!group) return { isSupported: false, gatewayReferenceId: null };

  const option = group.options.find(
    (opt) => opt.payment_method?.toUpperCase() === paymentMethod.toUpperCase()
  );
  if (!option) return { isSupported: false, gatewayReferenceId: null };

  const referenceIds = option.supported_reference_ids || [];
  if (!referenceIds.length) return { isSupported: false, gatewayReferenceId: null };

  return { isSupported: true, gatewayReferenceId: referenceIds[0] };
}
```

## Card BIN Lookup

When a user enters a card number, use the first 9 digits for a BIN lookup to determine the card network and validate support:

```typescript
// Debounce with ~300ms delay, trigger after 9+ digits entered
async function onCardNumberInput(cardNumber: string) {
  const digits = cardNumber.replace(/\s/g, "");
  if (digits.length < 9) return;

  const { data, error } = await sdk.payments.getCardInfo({
    cardbin: digits.slice(0, 9),
  });
  if (error) return;

  const cardInfo = data?.card_info;
  if (cardInfo?.brand) {
    // BIN lookup returns "MASTERCARD" but payment methods use "MASTER"
    const normalized = normalizeCardBrand(cardInfo.brand);
    const support = checkPaymentMethodSupport(groupedMethods, "CARD", normalized);
    // Show/hide "card not supported" message based on support.isSupported
  }
}

function normalizeCardBrand(brand: string): string {
  const map: Record<string, string> = {
    MASTERCARD: "MASTER",
    "AMERICAN EXPRESS": "AMEX",
    AMERICANEXPRESS: "AMEX",
  };
  return map[brand.toUpperCase()] || brand.toUpperCase();
}
```

## UPI VPA Validation

For UPI Collect payments, validate the VPA (Virtual Payment Address) before submitting:

```typescript
// Debounce with ~500ms delay
async function validateVpa(vpa: string) {
  // Local format check first (must contain @)
  if (!vpa.includes("@")) {
    showError("Invalid UPI ID format");
    return;
  }

  const { data, error } = await sdk.payments.verifyVpa({
    vpa: vpa.trim().toLowerCase(),
  });

  const isValid = data?.status === "valid";
  // Show validation result in UI
}
```

## Building Payment Payloads

The `payment_method` parameter for `createOrder` and `retryOrderPayment` varies by payment type. The `gateway_reference_id` comes from `checkPaymentMethodSupport()` above.

### Hyper Checkout (simplest — redirects to hosted payment page)

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "hyper-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  action: "paymentPage",
}
```

### Express Checkout — New Card

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "express-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  payment_method_type: "CARD",
  payment_method: "VISA", // from BIN lookup → normalizeCardBrand()
  auth_type: "OTP",
  card_number: "4111111111111111",
  card_exp_month: "12",
  card_exp_year: "2026",
  name_on_card: "Jane Doe",
  card_security_code: "123",
  save_to_locker: true,
}
```

### Express Checkout — Saved Card Token

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "express-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  payment_method_type: "CARD",
  payment_method: "VISA",
  auth_type: "OTP",
  card_token: savedCard.card_token,
  card_security_code: cvv,
}
```

### Express Checkout — UPI Collect

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "express-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  payment_method_type: "UPI",
  payment_method: "UPI_COLLECT",
  upi_vpa: "user@upi",
}
```

### Express Checkout — UPI Intent (opens UPI app)

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "express-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  payment_method_type: "UPI",
  payment_method: "UPI",
  txn_id: transactionId,
}
```

### Express Checkout — Net Banking

```typescript
{
  payment_provider_slug: "juspay",
  integration_type: "express-checkout",
  gateway_reference_id: gatewayRefId,
  return_url: `${window.location.origin}/order/confirm`,
  payment_method_type: "NB",
  payment_method: "NB_HDFC",
}
```

## Payment Status Polling

After creating an order, poll `getPaymentStatus` until the payment resolves:

```typescript
async function pollPaymentStatus(orderNumber: string) {
  const MAX_POLLS = 40;       // ~2 minutes at 3s intervals
  const POLL_INTERVAL = 3000;

  for (let i = 0; i < MAX_POLLS; i++) {
    // Note: getPaymentStatus takes a string, not an object
    const { data, error } = await sdk.order.getPaymentStatus(orderNumber);

    if (error) {
      await delay(POLL_INTERVAL);
      continue;
    }

    switch (data.status) {
      case "success":
        return showOrderConfirmation(orderNumber);
      case "failed":
        return data.is_retry_available
          ? showRetryPayment(orderNumber)
          : showPaymentFailed(orderNumber);
      case "pending":
      case "partially_paid":
        await delay(POLL_INTERVAL);
        continue;
    }
  }

  showPaymentPending(orderNumber); // Timed out
}
```

## Retry Payment

When payment fails and `is_retry_available` is `true`, retry with the same or different payment method:

```typescript
const { data, error } = await sdk.order.retryOrderPayment(
  { order_number: orderNumber },
  {
    payment_method: {
      // Same payment_method shapes as createOrder (see above)
      payment_provider_slug: "juspay",
      integration_type: "hyper-checkout",
      gateway_reference_id: gatewayRefId,
      return_url: `${window.location.origin}/order/confirm`,
      action: "paymentPage",
    },
  }
);
// Use data.payment_info to proceed with the retry payment flow
```

## Direct OTP Authentication (Express Checkout)

For express checkout card payments, the gateway may support direct OTP verification instead of a 3DS redirect. This keeps the user in your UI:

```typescript
// After createOrder with express-checkout card payment:
// 1. Check if response contains direct OTP challenge
// 2. Show OTP input in your UI
// 3. Verify:
await sdk.payments.authenticateDirectOtp({
  txn_id: transactionId,
  challenge_id: challengeId,
  otp: userEnteredOtp,
});

// 4. Start polling getPaymentStatus after OTP verification

// If user needs to resend:
await sdk.payments.resendDirectOtp({
  txn_id: transactionId,
  challenge_id: challengeId,
});
```

If direct OTP is not supported, fall back to a 3DS redirect (open the `auth_url` from the payment response in an iframe or redirect).

## SDK Methods Reference

All payment-related SDK methods used in custom checkout:

| Method | Client | Purpose |
|--------|--------|---------|
| `listPaymentMethods({ payment_provider_slug })` | `sdk.payments` | Available payment methods |
| `getCardInfo({ cardbin })` | `sdk.payments` | Card BIN lookup |
| `verifyVpa({ vpa })` | `sdk.payments` | UPI VPA validation |
| `listCustomerCards({ customer_id })` | `sdk.customer` | Saved cards |
| `listSavedPaymentMethods({ customer_id })` | `sdk.customer` | Saved UPI/wallets |
| `createOrder({ cart_id, payment_method? })` | `sdk.order` | Create order + initiate payment |
| `getPaymentStatus(orderNumber)` | `sdk.order` | Poll payment result (string param) |
| `retryOrderPayment({ order_number }, { payment_method })` | `sdk.order` | Retry failed payment |
| `authenticateDirectOtp({ txn_id, challenge_id, otp })` | `sdk.payments` | Direct OTP verification |
| `resendDirectOtp({ txn_id, challenge_id })` | `sdk.payments` | Resend OTP |
