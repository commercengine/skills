# Tool Registry

Every tool, what gates it, and the shapes it returns.

## Registry

### Catalog — always registered

| Tool | Purpose |
|------|---------|
| `search_products` | Keyword search. Returns product and variant identifiers, price, availability, storefront URL. |
| `get_product` | Full product details including variants, by product ID or storefront slug. |
| `browse_store` | Lists categories, or the products in one category. Returns storefront URLs. |
| `get_variant` | Resolves one variant to its canonical URL. Does **not** navigate. |

All four are annotated `readOnlyHint: true` and `untrustedContentHint: true` — the latter marks catalog copy as merchant-controlled text an agent should not treat as instructions.

### Cart — requires `session`

| Tool | Semantics |
|------|-----------|
| `get_cart` | Current lines with product/variant IDs, quantity, price, ordering rules. |
| `add_to_cart` | Quantities **add**. Up to 10 items per call. |
| `set_cart_item_quantity` | **Absolute** paid quantity. `0` removes the line. |
| `remove_from_cart` | Removes a line. Promo-added free lines are `removable: false`. |

### Checkout — requires `checkout`

| Tool | Gate |
|------|------|
| `open_cart` | `checkout` |
| `open_checkout` | `checkout` |
| `open_login` | `checkout.openLogin` |

### Navigation, session, content

| Tool | Gate |
|------|------|
| `open_product` | `navigation` |
| `manage_orders` | `navigation.ordersUrl` |
| `get_session_state` | `session` |
| `search_shop_policies_and_faqs` | `shopContent` |

## Bridges

```typescript
interface CommerceNavigationBridge {
  navigate(url: string): void | Promise<void>;
  ordersUrl?: string;                    // omit → no manage_orders tool
  allowedExternalOrigins?: string[];     // opt-in per origin; never inferred from siteUrl
}

interface CommerceCheckoutBridge {
  isReady(): boolean;
  openCart(): void | Promise<void>;
  openCheckout(): void | Promise<void>;
  openLogin?(): void | Promise<void>;    // omit → no open_login tool
}

interface CommerceShopContentBridge {
  searchPoliciesAndFaqs(query: string, signal?: AbortSignal): unknown | Promise<unknown>;
}
```

For hosted checkout, `createHostedCheckoutBridge` builds the checkout bridge from the store and wires `openLogin` only when the checkout build actually exposes it.

## Result Shape

Every tool returns a discriminated result — never a thrown exception, and never a raw SDK error object:

```typescript
type CommerceAiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: CommerceAiErrorCode; message: string; retryable?: boolean } };
```

`retryable` is the field agents act on. It is derived from what the call actually does:

| Call kind | Retryable on failure? |
|-----------|----------------------|
| `read` | Yes — repeating is free |
| `idempotent` (set quantity, remove) | Yes — repeating converges |
| `cumulative` (add to cart) | **No** — a retry would add twice |

That distinction is the difference between a helpful retry and a doubled order.

Upstream messages are capped so a verbose API error cannot flood the agent's context.

## Session Ownership

The Storefront SDK owns the session. The agent tools read it; they never mint one.

- `session` defaults to the storefront's own browser session. Pass an explicit client to override, or `null` to register no cart tools.
- `get_session_state` reports whether the shopper is signed in, anonymous, and whether login is even available on this storefront.
- **Cart merge on login is guaranteed only while the token survives** — the same browser session. Logging in on another device or browser will not merge an anonymous cart.

## Extending

Tools are grouped into modules. Add a capability without forking:

```typescript
import { createCommerceAiTools, type CommerceAiToolModule } from "@commercengine/ai";

const giftingModule: CommerceAiToolModule = {
  name: "gifting",
  build(context) {
    if (!context.checkout) return [];   // gate on a capability, like the built-ins
    return [{
      name: "add_gift_note",
      description: "Attach a gift note to the current cart.",
      inputSchema: {
        type: "object",
        additionalProperties: false,
        required: ["note"],
        properties: { note: { type: "string", maxLength: 500, description: "The note text." } },
      },
      annotations: { readOnlyHint: false },
      async execute(input, options) { /* ... */ },
    }];
  },
};

registerCommerceWebMcp({ ...config, extraModules: [giftingModule] });
```

- `extraModules` appends to the defaults — the common case.
- `modules` replaces them outright, to drop a capability or reorder tools.

A module returning `[]` is how gating works: no capability, no tool, no error.

## Writing Descriptions Agents Can Use

The built-in tools follow these rules; match them in your own modules.

- **State constraints in the description**, not only in the schema. `add_to_cart` names the min/max/increment rules so the agent gets it right first time instead of learning by rejection.
- **Cross-reference related tools.** `get_variant` says *"Does not navigate; use `open_product` to navigate."*
- **Say what the agent cannot do.** `open_login` says *"You cannot sign anyone in — never ask for a password or one-time code."*
- **Describe every parameter.** A bare `{ type: "string" }` makes the agent guess.
- **Set `readOnlyHint`** truthfully. Agents use it to decide what is safe to call while exploring; getting it wrong means either timid agents or unwanted mutations.
- **Return values, not strings.** The user agent serializes the result. Returning JSON text hands the agent something to re-parse.
