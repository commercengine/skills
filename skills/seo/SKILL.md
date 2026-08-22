---
name: ce-seo
description: SEO and agent-readable content for Commerce Engine storefronts using @commercengine/seo. Covers head metadata, JSON-LD (Product/ProductGroup with variesBy), canonical URLs, robots.txt, sitemaps, llms.txt, and Markdown mirrors across React SPA, Next.js, Astro, SvelteKit, and TanStack Start.
license: MIT
metadata:
  author: commercengine
  version: "1.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Commerce Engine SEO

`@commercengine/seo` turns a Commerce Engine catalog into everything a crawler or an AI agent needs: `<head>` metadata, JSON-LD, canonical URLs, `robots.txt`, sitemaps, `llms.txt`, and a Markdown mirror of every product and category.

Install alongside `@commercengine/storefront`:

```bash
npm install @commercengine/seo
```

## Impact Levels

- **CRITICAL** — wrong canonical/host, duplicate head tags, tools that never run
- **HIGH** — common wiring mistakes that typecheck and build cleanly
- **MEDIUM** — maintainability and correctness of generated output

## The One Decision That Matters

Everything else follows from this. **Ignore the framework; ask whether a server runs at request time.**

```
Does a server run on every request in production?
    │
    ├─ YES  (Next.js on a server, Astro SSR, SvelteKit with a Node/Vercel adapter,
    │        TanStack Start, any app behind a runtime)
    │        → Mount ONE request handler. Assets are generated on demand.
    │
    └─ NO   (React SPA, Astro static output, SvelteKit adapter-static,
             any prerendered/CDN-only build)
             → Run ONE prebuild script that writes real files into the
               published directory (public/ or static/).
```

Framework choice does **not** decide this — deployment mode does. A SvelteKit app can be either. Pick the row that matches how the app is actually served.

**Why a static build cannot use routes:** framework routers give a concrete route priority over a catch-all, so a `/product/[slug]` page claims `/product/shoe.md` before any catch-all sees it. A file in the published directory is not subject to routing at all.

## References

| Reference | When to Use |
|-----------|-------------|
| `references/head-and-jsonld.md` | CRITICAL — rendering head data per framework, JSON-LD, breadcrumbs, duplicate-tag rules |
| `references/deployment-modes.md` | CRITICAL — the server-mount and prebuild recipes, per framework, copy-paste ready |
| `references/routes-and-config.md` | HIGH — `defineCommerceSeoConfig`, the bundle boundary, custom route shapes, CMS slugs |
| `references/pitfalls.md` | CRITICAL — every failure mode observed in real integrations, with the symptom that gives it away |

## Two Modules, Never One

This is the single most important structural rule and it is invisible until something inspects your bundle.

`createCommerceSeo` holds a storefront client. Any module that exports the instance is **server-bound**. If that same module also exports `site` or `routes`, importing it from client code — to register agent tools, to build a canonical link — pulls the storefront into the browser bundle. It typechecks, it builds, and nothing tells you.

```typescript
// commerce-seo.config.ts — importable from ANYWHERE (client or server)
import { defineCommerceSeoConfig } from "@commercengine/seo/config";

export const commerceSeo = defineCommerceSeoConfig({
  site: {
    name: "Acme",
    url: "https://acme.example",     // origin only — no path, no trailing segment
    brandName: "Acme",
    description: "Small-batch goods.",
    locale: "en_US",
  },
  // The package default is "/products" (plural). This storefront serves PDPs at
  // /product/:slug, so it says so — a convention, not the default.
  routes: { productBase: "/product", categoryBase: "/category" },
});

export const { site, routes } = commerceSeo;
```

```typescript
// seo.ts — SERVER-BOUND. Never import this from a client component.
import { createCommerceSeo } from "@commercengine/seo";
import { commerceSeo } from "./commerce-seo.config";
import { serverStorefront } from "./storefront";

export const seo = createCommerceSeo({ ...commerceSeo, storefront: serverStorefront });
```

The result of `defineCommerceSeoConfig` is also structurally accepted by `@commercengine/ai`, so **one declaration configures both packages** — a crawler and an agent can never be told different URLs for the same product.

> `site.url` must be a bare origin. `createCommerceSeo` throws a `RangeError` on a sub-path like `https://acme.example/shop`.

## Quick Start by Framework

| Framework | Head / Metadata | Assets (server mode) | Assets (static mode) |
|-----------|-----------------|----------------------|----------------------|
| Next.js | `createProductMetadata` / `createCategoryMetadata` from `/nextjs` | `createNextjsSeoProxy` in `proxy.ts` | prebuild → `public/` |
| Astro | `createAstroProductHead` from `/astro` | `createAstroSeoMiddleware` from `/astro/server` | prebuild → `public/` |
| SvelteKit | `createSvelteKitProductHead` from `/sveltekit` | `createSvelteKitSeoHandle` from `/sveltekit/server` | prebuild → `static/` |
| TanStack Start | `createTanStackStartProductHead` from `/tanstack-start` | `createTanStackStartSeoMiddleware` from `/tanstack-start/server` | n/a (always server) |
| React SPA | `createProductHead` from the root export | n/a | prebuild → `public/` |

Full copy-paste recipes live in `references/deployment-modes.md`.

## What You Get

One `createCommerceSeo` instance exposes everything:

```typescript
await seo.productHead(product);      // title, meta, links, JSON-LD for a PDP
await seo.categoryHead(category);    // same for a PLP
await seo.productJsonLd(product);    // just the schema object
await seo.productUrl(product);       // resolved absolute URL, or null if omitted
await seo.categoryUrl(category);
await seo.productMarkdown(product);  // the .md mirror body
await seo.llmsTxt(categories);
seo.breadcrumbJsonLd([...]);         // synchronous
seo.config.site.url;                 // the configured origin
```

### Generated assets

| Path | Purpose |
|------|---------|
| `/robots.txt` | Crawl policy; points at the sitemap |
| `/sitemap.xml` | Products + categories, sharded past 50k URLs |
| `/llms.txt` | Store summary and category index for AI agents |
| `/sitemap.md` | Markdown index of every `.md` mirror |
| `{productBase}/{slug}.md` | Markdown mirror of a PDP |
| `{categoryBase}/{slug}.md` | Markdown mirror of a PLP |

The mirror paths follow your configured route bases, defaulting to `/products/{slug}.md` and `/category/{slug}.md`. Set `productBase` once and structured data, canonicals, `.md` alternates, `llms.txt`, sitemaps and the request handler's own matching all move together — there is no second place to keep in sync.

In server mode the same paths are also content-negotiated: a request for a PDP URL with `Accept: text/markdown` returns the mirror, with `Vary: Accept` set.

## Indexability

`indexable` is unset by default, which means **every page emits `noindex`**. That is deliberate — a staging or preview deployment should never be indexed. Set it explicitly when a real domain goes live:

```typescript
createCommerceSeo({ ...commerceSeo, storefront, indexable: true });
```

Omit it and the package falls back to `detectDeploymentEnvironment()`, which treats Vercel/Netlify preview deployments as non-production. If pages are unexpectedly `noindex` in production, this is why.

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Canonical points at a host that does not resolve | The configured `site.url` is authoritative for canonicals, sitemaps, breadcrumbs, and `.md` alternates. Verify the host resolves (`dig +short`) before shipping — a wrong domain is invisible in a build and poisons every URL at once. |
| CRITICAL | Duplicate `og:*` tags; crawler reads the wrong one | React, Astro, and SvelteKit do **not** deduplicate head content against static markup in `index.html` / `app.html` / a layout. Page-level tags coexist with sitewide defaults, and crawlers read the first. See `references/head-and-jsonld.md`. |
| CRITICAL | `.md` routes 404 in a static build | A concrete `/product/[slug]` route outranks any catch-all. Static builds must use the prebuild script, not routes. |
| CRITICAL | Two product schemas on one page | The package emits `Product`/`ProductGroup`. Delete any hand-rolled product JSON-LD — keeping both describes the same item twice. |
| HIGH | `BreadcrumbList` missing | `productHead` emits the **product schema only**. Breadcrumbs are the page's job via `seo.breadcrumbJsonLd([...])`. |
| HIGH | Breadcrumb/product URLs built by string concatenation | Use `seo.productUrl()` / `seo.categoryUrl()`. Concatenation silently ignores a custom `productBase`/`categoryBase` or a route resolver. |
| HIGH | `getProductDetail` result typed as `Product` | It returns `ProductDetail`, which is wider. The mistype is invisible until you pass it to the package. Fix the type at its source, not the call site. |
| HIGH | Client bundle pulls in the storefront | Split `commerce-seo.config.ts` from `seo.ts`. See "Two Modules, Never One". |
| MEDIUM | Prebuild script fails on CI with `ENOENT: .env.local` | `.env.local` is local-only; CI injects credentials into the environment. Make the file optional and fall back to `process.env`. |
| MEDIUM | Sitemap lists products the storefront cannot render | Return `null` from `routes.product` to omit a product; the sitemap, `llms.txt`, and category listings all honour it. |

## See Also

- `ce-ai` — WebMCP agent tools; shares `defineCommerceSeoConfig`
- `ce-catalog` — `Product` vs `Item`, variants, `has_variant`
- `ce-ssr-patterns` — `publicStorefront()` / `serverStorefront()` accessors
- `ce-setup` — SDK install and environment variables
