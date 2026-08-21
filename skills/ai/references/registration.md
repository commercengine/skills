# Registration

Where to mount `registerCommerceWebMcp` per framework, and — more importantly — how to prove it ran.

## Contract

```typescript
const registration: AbortController | null = await registerCommerceWebMcp(config);
```

- Returns `null` when the browser has no model context. Never throws for that case.
- Aborting the controller unregisters **everything** the call installed.
- A failure part-way through aborts too, so a partial tool set is never left behind.
- Safe to call more than once — abort the previous registration first.

## Mount Points

### Next.js — a client component in the root layout

```tsx
"use client";
import { registerCommerceWebMcp } from "@commercengine/ai/webmcp";
import { createHostedCheckoutBridge } from "@commercengine/ai/checkout";
import { getCheckout } from "@commercengine/checkout";
import { useRouter } from "next/navigation";
import { useEffect } from "react";
import { routes, site } from "@/lib/commerce-seo.config";
import { storefront } from "@/lib/storefront";

export function AgentTools() {
  const router = useRouter();

  useEffect(() => {
    let registration: AbortController | null = null;
    let cancelled = false;

    void registerCommerceWebMcp({
      storefront,
      siteUrl: site.url,
      routes,
      checkout: createHostedCheckoutBridge({ getState: () => getCheckout() }),
      navigation: { navigate: (url) => router.push(url) },
      diagnostics: process.env.NODE_ENV === "development"
        ? (event) => console.info("[commerce-ai]", event.code, event.message ?? "")
        : undefined,
    }).then((controller) => {
      if (cancelled) controller?.abort();
      else registration = controller;
    });

    return () => { cancelled = true; registration?.abort(); };
  }, [router]);

  return null;
}
```

Render `<AgentTools />` in the root layout. Import `@/lib/commerce-seo.config` — **not** the module holding the SEO instance, which is server-bound.

### TanStack Start — an effect in a root-level component

Same shape as Next: static import, `useEffect`, navigate via `router.navigate({ href: url })`.

### Astro — the client runtime, on `astro:page-load`

```typescript
// src/lib/client-runtime.ts
document.addEventListener("astro:page-load", () => void registerAgentTools());
document.addEventListener("astro:before-swap", () => agentTools?.abort());
```

View transitions swap the document, so re-register per page load and abort before the swap.

### SvelteKit — `onMount` in the root layout

```svelte
<script lang="ts">
let agentTools: AbortController | null = null;

onMount(() => {
  let cancelled = false;

  void (async () => { /* ...storefront bootstrap... */ })();

  // Registered alongside the bootstrap rather than after it: registration only declares
  // the tools, and each reports a retryable failure if called before the session is ready.
  void (async () => {
    const [{ registerCommerceWebMcp }, { createHostedCheckoutBridge }, { getCheckout }] =
      await Promise.all([
        import("@commercengine/ai/webmcp"),
        import("@commercengine/ai/checkout"),
        import("@commercengine/checkout"),
      ]);
    agentTools = await registerCommerceWebMcp({ /* ... */ });
  })();

  return () => { cancelled = true; agentTools?.abort(); };
});
</script>
```

> **Everything after `return` inside `onMount` is unreachable.** Putting the registration block below the cleanup means it never runs — and typecheck, build, and lint all still pass. Keep the `return` last.

### React SPA — after storefront init, or alongside it

Registration does not need a session. Gating it behind bootstrap only delays availability.

## Timing

Registration is not instant, and there is no readiness event an agent can wait on. Measured on a real deployment:

```
DOMContentLoaded          1022 ms
anonymous session         1027 → 1179 ms
lazy webmcp chunk           ~1469 ms
tools registered          after that
```

Consequences worth designing around:

- **A probe at page load finds nothing.** Agents that check once, early, will report no WebMCP. Lighthouse audits after the page settles, so it sees the tools and cannot distinguish 200 ms from 1.5 s.
- **Static imports beat dynamic ones** if you want the tools early. Three lazy `import()`s add a full network round trip.
- **Do not gate on the session.** Each tool already reports a retryable failure when called before the session or checkout is ready.

## Verifying It Actually Ran

The failure that matters is registration silently not happening. Reading the file is not sufficient — dead code looks fine.

**Use the diagnostic.** With `diagnostics` wired in dev, load the page and check the console:

```
[commerce-ai] unsupported No document.modelContext. WebMCP is available in Chromium 149+ …
```

That line means registration **ran**. `unsupported` is about the browser, not about your wiring — it is a success signal for the mount. No line at all means the code never executed.

Do not use these instead:

| Check | Why it fails |
|-------|--------------|
| Chunk name contains `webmcp` | SvelteKit and other bundlers emit opaque hashed names (`BmH9c5Jz.js`) |
| Typecheck / build / lint | Dead code is valid code |
| Lighthouse on one app per framework | Catches framework-layer bugs, misses per-app wiring |
| `document.modelContext` in a normal browser | Absent without the flag, whether or not you registered |

With the flag enabled you can confirm the surface directly:

```javascript
const tools = await document.modelContext.listTools?.();
console.log(tools.map((t) => t.name));
```

Note the agent-side API is string-flavoured — `executeTool(tool, jsonString)` takes arguments as a JSON string and returns `inputSchema` as a string. That is Chrome's shape, not the package's: `registerTool` receives `inputSchema` as an object and `execute` receives parsed input.
