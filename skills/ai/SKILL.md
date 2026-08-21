---
name: ce-ai
description: WebMCP agent tools for Commerce Engine storefronts using @commercengine/ai. Exposes catalog search, product/variant lookup, real cart mutations, navigation, and session state to browser AI agents via document.modelContext, across React SPA, Next.js, Astro, SvelteKit, and TanStack Start.
license: MIT
metadata:
  author: commercengine
  version: "1.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Commerce Engine Agent Tools (WebMCP)

`@commercengine/ai` publishes your storefront's capabilities to a browser AI agent through **WebMCP**. The agent searches the catalog, resolves variants, edits the shopper's real cart, and navigates the site — through the SDK, not by clicking DOM elements.

```bash
npm install @commercengine/ai
```

Peer: `@commercengine/storefront`. Optional: `@commercengine/checkout` (>= 0.3.0) for cart-drawer and checkout tools.

## Impact Levels

- **CRITICAL** — tools that never register, unbounded mutations, cross-origin navigation
- **HIGH** — capability gating, retry semantics, session ownership
- **MEDIUM** — diagnostics, timing, agent ergonomics

## Where the Boundary Sits

The package deliberately stops short of completing a purchase.

```
Agent CAN:  search · inspect products and variants · read the cart
            add / update / remove cart lines · navigate · open cart
            open checkout · open the login screen
Agent CANNOT: enter addresses · apply payment · place an order
            sign anyone in · read or supply a password or OTP
```

`open_checkout` hands control back to the shopper. There are no payment tools and no credential tools, and the login tool only *opens the screen* — its description tells the agent explicitly never to ask for a password or one-time code. This matches where every serious headless-commerce agent integration draws the line: an agent may assemble an order, a human completes it.

## Quick Start

```typescript
import { registerCommerceWebMcp } from "@commercengine/ai/webmcp";
import { createHostedCheckoutBridge } from "@commercengine/ai/checkout";
import { getCheckout } from "@commercengine/checkout";
import { routes, site } from "./commerce-seo.config";  // shared with @commercengine/seo
import { storefront } from "./storefront";

const registration = await registerCommerceWebMcp({
  storefront,
  siteUrl: site.url,
  routes,
  checkout: createHostedCheckoutBridge({ getState: () => getCheckout() }),
  navigation: { navigate: (url) => router.push(url) },
  diagnostics: import.meta.env.DEV
    ? (event) => console.info("[commerce-ai]", event.code, event.message ?? "")
    : undefined,
});

// Aborting unregisters everything this call installed.
registration?.abort();
```

Returns `null` — never throws — when the browser has no model context. Framework-specific mount points are in `references/registration.md`.

> `routes` and `site` come from `defineCommerceSeoConfig`. Sharing one declaration with `@commercengine/seo` means a crawler and an agent can never be told different URLs for the same product.

## References

| Reference | When to Use |
|-----------|-------------|
| `references/registration.md` | CRITICAL — where to mount per framework, timing, verifying it actually ran |
| `references/tools.md` | HIGH — the full tool registry, capability gating, result and error shapes |
| `references/pitfalls.md` | CRITICAL — failure modes, cart semantics, security boundaries |

## Capability Gating

Tools register **only** when you supply the capability behind them. A storefront with no checkout bridge simply has no cart-drawer tools; nothing errors.

| Config you pass | Tools you get |
|-----------------|---------------|
| *(always)* | `search_products`, `get_product`, `browse_store`, `get_variant` |
| `session` (defaults to the storefront's browser session) | `get_cart`, `add_to_cart`, `set_cart_item_quantity`, `remove_from_cart`, `get_session_state` |
| `checkout` | `open_cart`, `open_checkout` |
| `checkout.openLogin` | `open_login` |
| `navigation` | `open_product` |
| `navigation.ordersUrl` | `manage_orders` |
| `shopContent` | `search_shop_policies_and_faqs` |

A typical storefront with hosted checkout registers **12** tools. Pass `session: null` to register no cart tools at all.

This is why an agent reporting "login isn't available on this site" is usually correct and intentional — the checkout build exposed no `openLogin`.

## Cart Semantics

Get these wrong and an agent will corrupt a real shopper's cart.

| Tool | Semantics |
|------|-----------|
| `add_to_cart` | Quantities **add** to what is already there. Up to 10 items per call. |
| `set_cart_item_quantity` | **Absolute** paid quantity. `0` removes the line. |
| `remove_from_cart` | Removes a line outright. |

Three behaviours the package handles for you:

- **Promotional free items.** Lines auto-added by an active promotion are reported `removable: false` and cannot be removed. `set_cart_item_quantity` operates on the *paid* quantity, leaving the free allocation alone.
- **Ordering rules.** `min_order_quantity`, `max_order_quantity`, and increment steps are validated before the call, and the constraints are stated in the tool description so the agent can get it right first time rather than by trial and error.
- **Stale checkout.** The next add or create automatically resets a stale checkout session.

## Diagnostics

Registration reports what happened, always. Silence is the one thing it will not do:

```typescript
diagnostics: (event) => console.info("[commerce-ai]", event.code, event.message ?? "")
```

| `code` | Meaning |
|--------|---------|
| `unsupported` | No model context — the ordinary case, not a fault |
| `registered` | One tool registered (`event.tool` names it) |
| `registered:N` | Summary after a successful pass |
| `rollback` | A failure part-way through; everything was unregistered |

`message` is always a printable string, so a single console line never requires narrowing `error` first.

**Wire diagnostics in development on every app.** Without them, "did registration happen?" is unanswerable without probing globals — and that silence is exactly what lets a registration bug survive typecheck, build, lint, and a Lighthouse audit.

## WebMCP Availability

WebMCP is a Community Group **draft**, not a standard. It is available in Chromium 149+ behind `chrome://flags/#enable-webmcp-testing`, or under an origin trial token.

The draft moved `modelContext` from `navigator` to `document`. The package prefers `document.modelContext` and falls back to `navigator.modelContext`, so both work. When an agent reports "this page doesn't expose WebMCP", the overwhelmingly likely cause is that the flag is off — nothing the page does can change that.

Everything draft-specific is confined to one module, so churn is a change to one file rather than to every tool.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Registration never runs; typecheck/build/lint all pass | Dead or unreachable code, or a mount that never renders. Verify with the diagnostic in the console — reading the file is not enough. See `references/registration.md`. |
| CRITICAL | Treating `add_to_cart` as absolute | It **adds**. Use `set_cart_item_quantity` to set a value, and `0` to remove. |
| CRITICAL | Navigation to an attacker-chosen origin | External destinations require explicit opt-in via `navigation.allowedExternalOrigins`. Permission is never inferred from `siteUrl`. |
| HIGH | Expecting `open_login` and not getting it | It registers only when the checkout bridge exposes `openLogin`. |
| HIGH | Gating registration behind session bootstrap | Registration only *declares* the tools; each reports a retryable failure if called before the session is ready. Waiting delays availability for no benefit. |
| HIGH | Assuming a session merge across devices | Cart merge on login is guaranteed only while the token survives — same browser session. A different device or browser will not merge. |
| MEDIUM | No diagnostics in dev | Add them. This is the difference between a five-second check and an unanswerable question. |
| MEDIUM | Re-serializing tool output | `execute` returns a **value**; the user agent serializes it. Returning JSON text hands the agent a string to re-parse. |

## See Also

- `ce-seo` — shares `defineCommerceSeoConfig`; the `.md` mirrors and `llms.txt` serve crawlers the same content
- `ce-cart-checkout` — hosted checkout, cart rules, promotions
- `ce-catalog` — `Product` vs `Item`, variants, `has_variant`
