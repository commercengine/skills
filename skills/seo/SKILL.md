---
name: ce-seo
description: SEO and agent-readable content for Commerce Engine storefronts using @commercengine/seo. Covers head metadata, JSON-LD (Product/ProductGroup with variesBy), canonical URLs, robots.txt, sitemaps, llms.txt, and Markdown mirrors across React SPA, Next.js, Astro, SvelteKit, and TanStack Start — including CMS-owned URLs, where route records map several landing pages onto one catalog product.
license: MIT
metadata:
  author: commercengine
  version: "1.1.0"
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

## The Two Decisions That Matter

Everything else follows from these. Neither is a framework question.

### 1. Who owns the URLs?

```
Who decides a product's public URL?
    │
    ├─ THE CATALOG   → routes: { productBase, categoryBase }
    │                  The CE slug is the URL. One page per product.
    │
    └─ A CMS         → routes: createCmsRoutes({ source })
                       Pages declare themselves. Several pages may sell one product,
                       and the CMS slug is not the catalog slug.
```

A CMS-fronted storefront needs three answers a resolver cannot give — which page is *rendering*,
which page to *link to*, and what *all* the pages are. Declaring route records answers all three from
one source. `references/cms-routes.md` is the whole recipe; skip it entirely for a catalog-owned
storefront.

**The symptom of getting this wrong is a live 404:** product cards build `/products/<ce-slug>` by
hand while the route only ever renders CMS slugs.

### 2. Does a server run at request time?

**Ignore the framework; ask how the app is actually served.**

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
| `references/routes-and-config.md` | HIGH — `defineCommerceSeoConfig`, the bundle boundary, custom route shapes |
| `references/cms-routes.md` | CRITICAL **when a CMS owns the URLs** — route records, primary selection, hints, `markdown: false`, client-side links. Ignore it otherwise. |
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

### Config is validated at construction, not on first use

`site.url`, `routes.search` and the route bases are checked when `createCommerceSeo` runs, because
every canonical, sitemap entry and agent-facing URL is built from them and a wrong one is invisible
until something crawls it. Each of these throws a `RangeError`:

| Value | Why it is refused |
|---|---|
| `https://acme.example/shop` | Sub-path deployments are unsupported — every emitted URL would drop `/shop` |
| `ftp://acme.example` | Every URL built from it inherits the scheme |
| `https://user:pass@acme.example` | Credentials would be published in a public sitemap |
| `acme.example` | Not an origin. Nothing can infer whether an unschemed host is http or https |
| `//evil.example`, `/\evil.example` as a base | A *network-path reference* — passes `startsWith("/")`, then resolves to another host |
| `/shop/../products`, a backslash, a lone `%` | URL parsing rewrites these, so the emitted URL and the route matcher would disagree |

The **parsed** origin is what the instance carries, so `" https://acme.example/ "` is accepted and
normalized to `https://acme.example` rather than passing construction and throwing on first use. Read
`seo.config.site.url`, never the raw env var — they are not the same string.

If the value comes from an env var that is sometimes a bare domain, normalize it in **one** helper
and have every other place import that helper. Two copies of the "add `https://www.`" rule is how a
site ends up with canonicals at `https://www.acme.example` and a sitemap at
`https://www.www.acme.example`.

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
await seo.productUrl(product);       // the ONE url to link to, or null
await seo.categoryUrl(category);
await seo.productMarkdown(product);  // the .md mirror body
await seo.llmsTxt(categories);
seo.breadcrumbJsonLd([...]);         // synchronous
seo.config.site.url;                 // the NORMALIZED origin — not the raw string you passed
seo.searchPath;                      // "/search", or null when the storefront declared none
```

**Resolve a context once per page.** Calling `productHead()` and `productJsonLd()` back to back
resolves it twice — firing `enrichProduct` and a route call per variant twice — and nothing warns
you. `productPage()` makes that structurally impossible:

```typescript
const page = await seo.productPage(product, hint);
page.head(); page.jsonLd(); page.markdown(); page.context;
```

Where a CMS owns the URLs, four more matter — all covered in `references/cms-routes.md`:

```typescript
await seo.productUrls(input);           // EVERY url, for discovery — not just the primary
await seo.categoryUrls(category);
await seo.routeSpace();                 // the declared space as absolute URLs, or null
await seo.routeManifest();              // the manifest itself, or null
await seo.productMarkdownAvailable(slug);   // loads the route space — build/server only
```

### Generated assets

| Path | Purpose | Server mode |
|------|---------|-------------|
| `/robots.txt` | Crawl policy; points at the sitemap | needs `robots: true` |
| `/sitemap.xml` | Products + categories, sharded past 50k URLs | needs `sitemap: true` |
| `/llms.txt` | Store summary and category index for AI agents | always |
| `/sitemap.md` | Markdown index of every `.md` mirror | always |
| `{productBase}/{slug}.md` | Markdown mirror of a PDP | always |
| `{categoryBase}/{slug}.md` | Markdown mirror of a PLP | always |

The mirror paths follow your configured route bases, defaulting to `/products/{slug}.md` and `/category/{slug}.md`. Set `productBase` once and structured data, canonicals, `.md` alternates, `llms.txt`, sitemaps and the request handler's own matching all move together — there is no second place to keep in sync.

In server mode the same paths are also content-negotiated: a request for a PDP URL with `Accept: text/markdown` returns the mirror, with `Vary: Accept` set.

**`robots` and `sitemap` default to `false` in server mode** so the handler cannot silently shadow a route your app already owns. Pass `{ robots: true, sitemap: true }` to the adapter unless you serve those two yourself — see `references/deployment-modes.md`. Static mode is the opposite: `writeCommerceSeoAssets` emits both by default, and they are switched off with `includeRobots: false` / `includeSitemapXml: false`.

## Indexability

`indexable` **overrides** deployment detection; it does not default to `false`.

```typescript
const indexable = config.indexable ?? isProductionDeployment(config.deployment);
```

- `indexable: true` — force indexing.
- `indexable: false` — force `noindex`.
- **omitted** — detect the deployment. A recognised production deployment is indexable; preview, development and unknown are not.

So a real production deploy on Vercel, Netlify or Cloudflare — or any host where `NODE_ENV=production` — is indexable *without* setting the flag. You do not need `indexable: true` in production as a matter of course.

| Platform | Signal | Production when |
|----------|--------|-----------------|
| Vercel | `VERCEL_ENV` | `production` |
| Netlify | `CONTEXT` | `production` |
| Cloudflare | branch name | branch ∈ `productionBranches` (default `main`, `master`) |
| anything else | `NODE_ENV` | `production` |

`NODE_ENV` alone cannot answer this, which is why detection exists: Vercel, Netlify and Cloudflare all build **preview** deployments with `NODE_ENV=production`. Trusting it would publish an indexable copy of every branch, competing with the real storefront.

**Unknown is treated as non-indexable on purpose.** An unrecognised host is more likely to be a preview or a local build than the canonical production site, and the cost is asymmetric: a missing `noindex` competes with your real store in search results, while an unnecessary one is fixed by setting the flag. On a platform outside the table, set `indexable` explicitly.

> A non-indexable deployment stays **crawlable** — its `robots.txt` is `User-agent: * / Allow: /` with no sitemap, and every page carries `noindex`. That combination is deliberate: `Disallow: /` stops the crawl, and a crawler that never fetches the page never reads the `noindex`, so the URL can still be indexed without content. Do not "harden" a preview by adding `Disallow: /`.

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
| CRITICAL | Product cards 404 on a CMS-fronted storefront | Cards build `/products/<ce-slug>` while the route renders CMS slugs. Resolve through the manifest and render a **non-link** when it returns `null` — never fall back to the catalog slug, which is the URL that 404s. See `references/cms-routes.md`. |
| CRITICAL | Five of six landing pages missing from the sitemap | A resolver returns one URL per product. Declare `routes.list` so discovery enumerates every page. |
| HIGH | Every landing page declares the same canonical | Rendering without a hint answers with the *primary*. Pass the hint on `productPage` / `productHead` / `createProductMetadata` so each page is canonical to itself. |
| HIGH | Page advertises a `.md` mirror that 404s | The page and the mirror are separate claims. Carry `markdown` on the hint from the CMS record the render already fetched, rather than probing the catalog per render. |
| MEDIUM | Prebuild script fails on CI with `ENOENT: .env.local` | `.env.local` is local-only; CI injects credentials into the environment. Make the file optional and fall back to `process.env`. |
| MEDIUM | Sitemap lists products the storefront cannot render | Return `null` from `routes.product` to omit a product; the sitemap, `llms.txt`, and category listings all honour it. |

## See Also

- `ce-ai` — WebMCP agent tools; shares `defineCommerceSeoConfig`
- `ce-catalog` — `Product` vs `Item`, variants, `has_variant`
- `ce-ssr-patterns` — `publicStorefront()` / `serverStorefront()` accessors
- `ce-setup` — SDK install and environment variables
