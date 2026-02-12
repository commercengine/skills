---
name: ce-subscriptions
description: Commerce Engine subscription management. Standard and custom plans, billing/shipping cycles, trial periods, pause/revoke, and coupon integration.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Subscription Management

> **Prerequisite**: Requires Juspay payment gateway integration for recurring payment mandates. Products must have subscription plans defined in their catalog data.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| Create subscription | `sdk.subscription.createSubscription({ ... })` |
| List subscriptions | `sdk.subscription.listSubscriptions()` |
| Get subscription detail | `sdk.subscription.retrieveSubscription(subscriptionId)` |
| Update subscription | `sdk.subscription.updateSubscription(subscriptionId, { command: "update", ... })` |
| Pause subscription | `sdk.subscription.updateSubscription(subscriptionId, { command: "pause", ... })` |
| Revoke subscription | `sdk.subscription.updateSubscription(subscriptionId, { command: "revoke", reason })` |

## Key Distinction

Subscriptions are **separate from the standard checkout flow**. They bypass the cart-to-order conversion. Creating a subscription uses `POST /subscriptions` directly, followed by a mandate/payment step.

## Plan Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Standard** | Pre-defined plan from product catalog | "Subscribe & Save" on product pages |
| **Custom** | Custom schedule defined at creation | Multi-item subscriptions, custom cycles |

## Subscription Statuses

| Status | Description |
|--------|-------------|
| `created` | Subscription created, pending first payment |
| `active` | Active and billing on schedule |
| `paused` | Temporarily paused, will resume |
| `revoked` | Permanently cancelled |

## Decision Tree

```
User Request
    │
    ├─ "Subscribe to product"
    │   ├─ Product has plans? → Check product.subscription[] array
    │   ├─ Standard plan → createSubscription({ plan_type: "standard", plan_id })
    │   └─ Custom plan → createSubscription({ plan_type: "custom", billing_frequency, ... })
    │
    ├─ "My subscriptions"
    │   └─ sdk.subscription.listSubscriptions()
    │
    ├─ "Pause subscription"
    │   └─ updateSubscription(id, { command: "pause", pause_end_date? })
    │
    ├─ "Cancel subscription"
    │   └─ updateSubscription(id, { command: "revoke", reason })
    │
    └─ "Change subscription"
        └─ updateSubscription(id, { command: "update", ... })
```

## Key Patterns

### Create Standard Subscription

```typescript
// Product catalog includes subscription plans:
// product.subscription = [{ id: "plan_123", subscription_plan: "Monthly", ... }]

const { data, error } = await sdk.subscription.createSubscription({
  plan_type: "standard",
  plan_id: "plan_123",           // From product.subscription[].id
  start_date: "2025-01-01",      // Optional, defaults to today
  coupon_code: "SUB10OFF",       // Optional
  invoice_items: [
    { product_id: "prod_123", variant_id: "var_456", quantity: 1 },
  ],
});
```

### Create Custom Subscription

```typescript
const { data, error } = await sdk.subscription.createSubscription({
  plan_type: "custom",
  billing_frequency: "monthly",
  billing_interval: 1,            // Every 1 month
  billing_limit: 12,              // Max 12 billing cycles (null for indefinite)
  shipping_frequency: "monthly",  // For physical products
  shipping_interval: 1,
  coupon_code: "SUBSCRIBE20",
  invoice_items: [
    { product_id: "prod_123", variant_id: "var_456", quantity: 2 },
    { product_id: "prod_789", quantity: 1 },
  ],
});
```

### List & View Subscriptions

```typescript
// List all subscriptions
const { data, error } = await sdk.subscription.listSubscriptions();

// Get detailed subscription info
const { data, error } = await sdk.subscription.retrieveSubscription(subscriptionId);

// data.subscription includes: status, next_billing_date, invoice_items,
// billing_address, shipping_address, coupon info, trial dates
```

### Pause Subscription

```typescript
const { data, error } = await sdk.subscription.updateSubscription(subscriptionId, {
  command: "pause",
  pause_start_date: "2025-03-01",  // Optional: schedule pause
  pause_end_date: "2025-04-01",    // Optional: auto-resume date (null = indefinite)
});
```

### Revoke (Cancel) Subscription

```typescript
// Check if cancellation is allowed
const { data: sub } = await sdk.subscription.retrieveSubscription(subscriptionId);

if (sub.subscription.is_cancellation_allowed) {
  const { data, error } = await sdk.subscription.updateSubscription(subscriptionId, {
    command: "revoke",
    reason: "No longer needed",
  });
}
```

### Update Subscription

```typescript
const { data, error } = await sdk.subscription.updateSubscription(subscriptionId, {
  command: "update",
  billing_frequency: "monthly",
  billing_interval: 2,          // Change to every 2 months
  invoice_items: [              // Update items
    { product_id: "prod_123", variant_id: "var_456", quantity: 3 },
  ],
});
```

## Billing & Shipping Cycles

| Field | Description | Example |
|-------|-------------|---------|
| `billing_frequency` | How often to bill | `monthly`, `weekly`, `daily` |
| `billing_interval` | Multiplier | `2` = every 2 months |
| `billing_limit` | Max cycles | `12` = 12 months, `null` = indefinite |
| `shipping_frequency` | How often to ship | Same options as billing |
| `shipping_interval` | Multiplier | Can differ from billing |
| `shipping_limit` | Max shipments | `null` = indefinite |

## Trial Periods

Available for digital products:

```typescript
// Trial configured in product subscription plan
// subscription.trial_days = 14
// subscription.trial_start_date / trial_end_date auto-calculated
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Missing Juspay integration | Juspay is required for recurring payment mandates |
| HIGH | Using cart checkout for subscriptions | Subscriptions use `POST /subscriptions`, not `createOrder()` |
| HIGH | Product missing subscription plans | Check `product.subscription[]` array — plans must be defined in catalog |
| MEDIUM | Not checking `is_cancellation_allowed` | Some subscriptions have `days_until_cancellation_allowed` restrictions |
| MEDIUM | Ignoring `coupon_redemption_limit` | Coupon may only apply for N cycles, not the full subscription |
| LOW | Indefinite pause without end date | Consider setting `pause_end_date` to auto-resume |

## See Also

- `catalog/` - Product subscription plan data
- `webhooks/` - Subscription lifecycle events (created, renewed, paused, revoked)
- `cart-checkout/` - One-time purchases (separate flow)

## Documentation

- **Subscriptions Guide**: https://www.commercengine.io/docs/storefront/subscriptions
- **API Reference**: https://www.commercengine.io/docs/api-reference/subscriptions
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/create-subscription
