---
name: ce-ai
description: WebMCP agent tools for Commerce Engine storefronts using @commercengine/ai. Exposes catalog search, product/variant lookup, real cart mutations, navigation, and session state to browser AI agents via document.modelContext, across React SPA, Next.js, Astro, SvelteKit, and TanStack Start. Covers registration lifecycle and cancellation, capability gating, and resolving CMS slugs in both directions.
license: MIT
metadata:
  author: commercengine
  version: "1.1.0"
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

// Keep the controller for teardown. Calling it unregisters every tool this call installed —
// so call it from a cleanup function, not here:
// registration?.abort();
```

Returns `null` — never throws — when the browser has no model context. Framework-specific mount points are in `references/registration.md`.

> `routes` and `site` come from `defineCommerceSeoConfig`. Sharing one declaration with `@commercengine/seo` means a crawler and an agent can never be told different URLs for the same product.

`siteUrl` is validated where it is configured, not carried into an agent's results: an `ftp://` scheme, embedded credentials, or a sub-path throws a `RangeError` at construction. An agent is the surface where a bad origin does the most damage, because every URL a tool returns is built from it and handed straight to a model.

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

Counted against the published package, so you can predict the number before you look:

| Configuration | Tools |
|---|---|
| `session: null`, nothing else | 4 |
| catalog + session *(the default)* | 9 |
| + hosted checkout | 11 |
| + `checkout.openLogin` | 12 |
| + `navigation` | 13 |
| + `navigation.ordersUrl` | 14 |
| + `shopContent` | 15 |

The session tools come from the storefront's own browser session, found via `clientStorefront()` or
`session()`. A storefront exposing neither registers the four catalog tools and nothing else — which
looks identical to a bug, so check the accessor name first when cart tools are missing.

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

## Two Signals, Doing Different Jobs

Claiming a tool set is **not atomic** — it is a sequence of awaited `registerTool` calls. Two
overlapping registrations against one model context would race for the same names and one would throw
`InvalidStateError: Duplicate tool name`. React Strict Mode makes this the ordinary case, not an edge
one: it mounts, cleans up and mounts again back-to-back on every dev mount, and both passes dispatch
their first tool before either resumes, so cleanup alone cannot break the tie.

Registrations against one context are therefore **serialized** — a second call waits for the first to
settle, including any rollback.

| Signal | Cancels | Why the other cannot cover it |
|---|---|---|
| the returned controller | tools that **did** register | it does not exist until the promise resolves |
| `config.signal` | a registration still **in progress**, including the `registerTool` currently awaited | the controller cannot reach a pass that has not finished |

```typescript
import { CommerceWebMcpAbortError, registerCommerceWebMcp } from "@commercengine/ai/webmcp";

useEffect(() => {
  const controller = new AbortController();
  let registration: { abort: () => void } | null = null;
  void registerCommerceWebMcp({ ...config, signal: controller.signal })
    .then((result) => { registration = result; })
    .catch((error) => { if (!(error instanceof CommerceWebMcpAbortError)) throw error; });
  return () => { controller.abort(); registration?.abort(); };
}, []);
```

Cancellation rejects with `CommerceWebMcpAbortError`, which callers are expected to swallow — it
means "you asked for this". Without `config.signal`, cleanup can run before the controller exists and
the stale pass stays live after its component has gone away.

## Routes: The Agent Resolves Both Directions

`routes.product` runs **in the browser**, during tool execution. A resolver that performs a CMS lookup
would need that credential on the client, so the portable shape is a pure function over route data
the browser already has.

```typescript
import { createCommerceRouteManifest } from "@commercengine/seo/routes";
const manifest = createCommerceRouteManifest(await loadRouteRecords());

export const routes = {
  // OUTBOUND: entity → URL. Never guess. `null` means "no public page".
  product: (input) => manifest.productPath(input),

  // INBOUND: public slug → CE identifier. The `?? publicSlug` tail is a deliberate
  // choice, valid ONLY because this storefront also answers to catalog slugs — it
  // keeps a page published since the last refresh working when the two agree.
  // Return null instead where CMS slugs are the only accepted form.
  resolveProductRoute: (publicSlug) => manifest.resolveProduct(publicSlug)?.productId ?? publicSlug,
};
```

`resolveProductRoute` matters as much as `product`. An agent reads a CMS page slug off the page and
calls `get_product("knee-pain-relief-oil")` — without the inbound mapping that goes straight to a
catalog which has never heard of it.

**The two directions have opposite fallback rules, and the distinction is easy to lose:**

| Direction | On no match | Why |
|---|---|---|
| Outbound `productPath()` | return `null`, render a non-link | a guessed `/products/${ce_slug}` is a shopper-facing 404 |
| Inbound `resolveProductRoute()` | `?? publicSlug` **only** if catalog slugs are also valid public URLs; otherwise `null` | the cost is one failed lookup, and the fallback rescues a page published since the last cache refresh |

Returning `null` from an inbound resolver fails the tool with `route_not_found` before any catalog
request, rather than passing an unmapped slug to a catalog that has never heard of it.

Separately: **do not ship a 100,000-product manifest to the browser** merely to resolve a search page.
This package performs no route-data transport and imposes no API, batching or cache policy — at that
size, back the same hooks with a storefront-owned point cache that loads only the result set.

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
| CRITICAL | Aborting in the quick start | `registration?.abort()` immediately after registering unregisters everything you just installed. It belongs in a cleanup function. |
| HIGH | `InvalidStateError: Duplicate tool name` in dev | Two overlapping registrations. Pass `config.signal` so the stale pass is cancelled rather than left live. See "Two Signals". |
| HIGH | Agent gets a 404 for a product that exists | The CMS slug reached the catalog untranslated. Supply `resolveProductRoute`. |
| MEDIUM | Cart tools missing with no error | The storefront exposes no `clientStorefront()` / `session()` accessor, so `session` resolved to `null`. Four catalog tools register and nothing complains. |

## See Also

- `ce-seo` — shares `defineCommerceSeoConfig`; the `.md` mirrors and `llms.txt` serve crawlers the same content
- `ce-cart-checkout` — hosted checkout, cart rules, promotions
- `ce-catalog` — `Product` vs `Item`, variants, `has_variant`
