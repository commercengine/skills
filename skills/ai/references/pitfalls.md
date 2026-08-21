# Pitfalls

Failure modes seen in real integrations. Symptom first — the symptom is rarely where the bug is.

## CRITICAL

### Tools never register, and everything reports success

**Symptom:** the site works, typecheck/build/lint pass, a Lighthouse audit on a sibling app detects WebMCP — and this app exposes nothing.

Observed cause: the registration block sat **after** the cleanup `return` inside `onMount`. Unreachable code is valid code, so nothing complained.

```javascript
onMount(() => {
  void bootstrap();
  return () => { cancelled = true; };
  void registerAgentTools();   // ← never runs
});
```

Other ways to get here: a component that is never rendered, an effect whose dependency array never satisfies, a conditional mount behind a flag that is off in production.

**Detect it with a runtime signal.** Wire `diagnostics` in dev, load the page, and look for the `[commerce-ai]` line. Its presence proves the code ran; `unsupported` is about the browser, not your wiring. No line means it never executed.

These do **not** detect it: chunk-name greps (bundlers emit opaque hashes), typecheck, build, lint, or a spot check on one app per framework.

### `add_to_cart` treated as absolute

**Symptom:** an agent "correcting" a quantity doubles it.

`add_to_cart` **adds** to what is already there. To set a value use `set_cart_item_quantity`; to remove, set it to `0`. This is also why `add_to_cart` is non-retryable: a retried add is a doubled line.

### Navigation to an arbitrary origin

Catalog values are merchant-controlled and reach the agent as content. If navigation trusted any URL it was handed, a compromised catalog field could redirect a shopper off-site.

External destinations require explicit opt-in:

```typescript
navigation: { navigate, allowedExternalOrigins: ["https://help.acme.example"] }
```

Permission is never inferred from `siteUrl`. Grant per origin, and only where you mean it.

### Removing a promotional free item

Lines auto-added by an active promotion cannot be removed while the promotion applies. They are reported `removable: false` so an agent can explain rather than retry.

`set_cart_item_quantity` targets the **paid** quantity. A line with 2 paid and 1 free, set to 1, leaves 1 paid and 1 free — not 1 total.

## HIGH

### Registration gated behind session bootstrap

Registration only *declares* the tools. Every tool already reports a retryable failure if called before the session or checkout is ready — `checkout_not_ready` exists precisely for this.

Gating on bootstrap costs a network round trip before an agent can see anything, for no correctness benefit. Register early; let the tools handle not-ready.

### Expecting a tool that was never configured

`open_login` requires `checkout.openLogin`. `manage_orders` requires `navigation.ordersUrl`. `search_shop_policies_and_faqs` requires `shopContent`. Cart tools require `session`.

An agent reporting "login isn't available on this site" is usually reporting the truth about your configuration.

### Assuming cart merge across devices

Merge is guaranteed only while the token survives — the same browser session. Same session: reliable. Different device or browser: it will not merge, and no tool can make it.

### Importing the server-bound SEO module for `site`/`routes`

```typescript
import { site, routes } from "./seo";                  // drags the storefront into the bundle
import { site, routes } from "./commerce-seo.config";  // storefront-free
```

Both typecheck and build. Only a bundle inspection tells them apart. See `ce-seo`.

### Raw SDK exceptions reaching the agent

Some SDK calls throw rather than returning `{ data, error }` — reading a cart with no session, for example. Wrap SDK calls so the agent gets a typed failure with a `code` and a `retryable` flag, not a stack trace. The built-in tools do this; custom modules must too.

## MEDIUM

### No diagnostics in development

The single highest-leverage line in the integration:

```typescript
diagnostics: import.meta.env.DEV
  ? (event) => console.info("[commerce-ai]", event.code, event.message ?? "")
  : undefined,
```

Without it, "are the tools live?" requires probing globals in a flag-enabled browser. With it, it is a glance at the console. The registration bug above survived every automated check precisely because the affected app passed no diagnostics.

`message` is always printable, so logging never requires narrowing `error` first.

### Expecting instant availability

There is no readiness event. Tools appear after hydration, plus any lazy chunks, plus registration. An agent probing at load finds nothing; Lighthouse, auditing after the page settles, finds everything. Both observations can be true at once.

### Re-serializing tool output

`execute` returns a **value**. The user agent serializes it. Returning JSON text hands the agent a string to re-parse.

### Assuming `navigator.modelContext`

The draft moved `modelContext` from `navigator` to `document`. The package prefers `document` and falls back to `navigator`. Check both before concluding a page has no WebMCP.

## Security Model

| Concern | Handling |
|---------|----------|
| Catalog text as instructions | Read tools carry `untrustedContentHint: true` |
| Off-site navigation | Opt-in per origin; never inferred from `siteUrl` |
| Credentials | No tool reads, accepts, or transmits a password or OTP |
| Payment | No payment tools exist; `open_checkout` hands control to the shopper |
| Unbounded mutation | Batch capped at 10 items; quantities validated against ordering rules before the call |
| Runaway retries | Non-idempotent calls are marked non-retryable |
| Verbose upstream errors | Messages truncated before reaching the agent |

## Verification Checklist

Before shipping agent tools on a storefront:

- [ ] `[commerce-ai]` diagnostic appears in the dev console — on **every** app, not one per framework
- [ ] Tool count matches the capabilities configured (12 for a typical hosted-checkout storefront)
- [ ] `site`/`routes` imported from the storefront-free config module
- [ ] `allowedExternalOrigins` lists only origins you intend
- [ ] With the flag on: add an item, confirm the cart drawer opens and the real cart updates
- [ ] Cleanup aborts the registration on unmount
