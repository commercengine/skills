# Routes & Configuration

## `defineCommerceSeoConfig`

Identity at runtime — it exists to give site identity and route shapes a typed, storefront-free home, and to make "which module may client code import?" answerable by looking at the import specifier.

```typescript
import { defineCommerceSeoConfig } from "@commercengine/seo/config";

export const commerceSeo = defineCommerceSeoConfig({
  site: {
    name: "Acme",                        // required
    url: "https://acme.example",         // required — bare origin, no path
    brandName: "Acme",                   // schema.org brand; defaults to `name`
    description: "Small-batch goods.",   // used in llms.txt and WebSite schema
    logoUrl: "https://acme.example/logo.png",
    locale: "en_US",
  },
  // Explicit because this storefront serves PDPs at /product/:slug.
  // The package default is /products/:slug — see "Route Shapes" below.
  routes: { productBase: "/product", categoryBase: "/category" },
});

export const { site, routes } = commerceSeo;
```

`@commercengine/seo/config` imports **nothing** from `@commercengine/storefront` at runtime. Its built graph is three modules (`config`, `routes`, `utils`), which is what makes it safe to import from a client component.

The same object is structurally accepted by `@commercengine/ai`:

```typescript
registerCommerceWebMcp({ storefront, siteUrl: site.url, routes, /* ... */ });
```

One declaration, two packages, no chance of a crawler and an agent disagreeing about a URL.

## Route Shapes

### Simple: change the base

```typescript
routes: { productBase: "/p", categoryBase: "/collections" }
```

The package defaults are:

```typescript
DEFAULT_PRODUCT_BASE  = "/products";   // → /products/:slug
DEFAULT_CATEGORY_BASE = "/category";   // → /category/:slug
DEFAULT_SEARCH_PATH   = "/search";     // the query is appended as ?q=…
```

Note `/products`, plural. Many Commerce Engine starters explicitly configure `productBase: "/product"` — that is an **application convention, not the package default**. If you omit `routes` entirely you get `/products/:slug`, so a storefront whose pages live at `/product/:slug` must say so.

### Custom: a function per entity

`routes.product` receives a `ProductRouteInput` and nothing else:

```typescript
interface ProductRouteInput {
  productId: string;
  productSlug: string;
  variantId?: string | null;
  variantSlug?: string | null;
}
```

```typescript
routes: {
  product: ({ productSlug, variantSlug }) =>
    variantSlug ? `/shop/${productSlug}/${variantSlug}` : `/shop/${productSlug}`,
  category: (category) => `/collections/${category.slug}`,
}
```

`routes.category` receives the whole `Category`, so `category.slug`, `category.id` and `category.name` are all available there.

There is **no category information in `ProductRouteInput`** — a product can belong to several categories, so there is no single correct answer for the package to supply. A URL shape like `/shop/:category/:product` therefore has to get that segment from your own data:

```typescript
routes: {
  product: async ({ productId, productSlug }) => {
    const category = await cms.primaryCategoryFor(productId);
    return `/shop/${category?.slug ?? "all"}/${productSlug}`;
  },
}
```

Return `null` to **omit** an entity entirely. Omission is honoured everywhere at once — sitemap, `llms.txt`, category listings, and the `.md` handler (which then 404s). Use it for unpublished or gated products rather than filtering in three places.

Both may be async, so a CMS lookup is fine.

### Inbound: mapping public slugs back to CE

If your public URLs use CMS slugs rather than Commerce Engine slugs, the outbound function alone is not enough — the `.md` handler needs to resolve an incoming public slug back to something the catalog understands:

```typescript
routes: {
  // Outbound: CE entity → public URL. Use productId, the stable identifier.
  product: async ({ productId }) => {
    const slug = await cms.slugForProduct(productId);
    return slug ? `/products/${slug}` : null;
  },
  // Inbound: public slug → something the catalog can look up.
  resolveProductRoute: async (publicSlug) => (await cms.lookup(publicSlug))?.ceProductId ?? null,
  resolveCategoryRoute: async (publicSlug) => (await cms.lookupCategory(publicSlug))?.ceId ?? null,
}
```

Without these, `/products/editorial-speaker.md` is looked up in the catalog by the literal string `editorial-speaker` and 404s.

### Always resolve, never concatenate

```typescript
const url = await seo.productUrl(product);      // honours productBase AND routes.product
const url = `${site.url}/product/${product.slug}`;  // silently wrong under a custom route
```

`productUrl` / `categoryUrl` return `null` for an omitted entity — fall back to `site.url` when you need a non-null string (breadcrumbs, for example).

## Enrichment Hooks

Override generated copy per entity without forking the package:

```typescript
createCommerceSeo({
  ...commerceSeo,
  storefront,
  enrichProduct: async (product) => ({
    title: cms.title(product.id) ?? undefined,
    description: cms.description(product.id) ?? undefined,
  }),
  enrichCategory: (category) => ({ description: copy[category.slug] }),
});
```

Each hook runs **once** per head build. Returning `{}` (or omitting a field) keeps the generated value.

## Indexability & Environment

`indexable` overrides detection. It is not a flag that defaults to `false`:

```typescript
// createCommerceSeo, effectively:
const indexable = config.indexable ?? isProductionDeployment(config.deployment);
```

```typescript
indexable: true       // force index
indexable: false      // force noindex
// omitted           → isProductionDeployment(deployment)
```

`detectDeploymentEnvironment()` returns `"production" | "preview" | "development" | "unknown"`, and only `"production"` is indexable:

| Platform | Signal | Production when |
|----------|--------|-----------------|
| Vercel | `VERCEL_ENV` | `production` |
| Netlify | `CONTEXT` (with `NETLIFY`) | `production` |
| Cloudflare Pages / Workers | `CF_PAGES_BRANCH` / `WORKERS_CI_BRANCH` | branch ∈ `productionBranches` (default `["main", "master"]`) |
| fallback | `NODE_ENV` | `production` |
| no signal | — | `unknown` → **not** indexable |

Three consequences worth internalising:

- **A recognised production deployment is indexable with no flag set.** `indexable: true` is an override, not a requirement for going live.
- **Unknown is conservative on purpose.** No platform signal is more likely to mean a preview or a local build than the canonical site, and the costs are asymmetric — a missing `noindex` competes with your real store, an unnecessary one is a one-line fix. On an unrecognised host, say so explicitly.
- **Cloudflare publishes only a branch name**, so if your production branch is not `main`/`master`, configure it or production serves the preview policy:

  ```typescript
  createCommerceSeo({ ...commerceSeo, storefront, deployment: { productionBranches: ["release"] } });
  ```

### A non-indexable deployment is still crawlable

This surprises people, so it is worth stating plainly. When `indexable` resolves false the generated `robots.txt` is:

```
User-agent: *
Allow: /
```

— no sitemap advertised, and `noindex, nofollow` on every page, `X-Robots-Tag` on every served document, and a `_headers` file for static output. It does **not** disallow anything.

`Disallow: /` would look safer and is actively counterproductive: it stops the crawl without deindexing, and a crawler that never fetches the page never reads the `noindex`, so a linked URL can still be indexed with no content. Deindexing requires the crawler to fetch and read the directive. Do not add `Disallow: /` to a preview.

Indexability is resolved **once** per `createCommerceSeo` and exposed as `seo.indexable`; `robots.txt` and the page directives both derive from it, so they cannot disagree. There is deliberately no per-call override.

`applyIndexability` is exported if you need the same decision elsewhere.

## Sitemaps

Sharding is automatic at `SITEMAP_URL_LIMIT` (50,000). Past that, `/sitemap.xml` becomes an index and shards live at `/sitemap/{n}.xml`. A request for a shard id that does not exist returns 404 rather than an empty sitemap.

Product pagination uses a fixed page size internally. Do not try to shrink it to fit a remaining budget — the API derives its offset from `page * limit`, so a variable page size re-reads records already seen and skips others.

For Next's `app/sitemap.ts` convention:

```typescript
import { createSitemap, createSitemapIds } from "@commercengine/seo/nextjs";
export const generateSitemaps = createSitemapIds(seo);
export default createSitemap(seo);
```

## Markdown Mirrors

Every product and category gets a `.md` twin: frontmatter (canonical URL, price, availability) plus a readable body. `llms.txt` summarises the store and indexes categories; `sitemap.md` indexes every mirror.

Two behaviours worth knowing:

- Category listings only link a `.md` mirror for products that actually have one. A product omitted by `routes.product` is listed without a dead link.
- Markdown escaping covers link and HTML syntax, so a product name containing `[`, `]`, `<`, or `|` cannot break the table or inject markup.

## Bundle Boundary Check

Prove the split rather than trusting review:

```bash
# No client chunk may contain the SEO instance or a server storefront.
grep -rl "createCommerceSeo" dist/assets/ .next/static/ 2>/dev/null
```

Expect no matches for a server-rendered app. For a single-page app the storefront legitimately ships to the browser — there the check is that `commerce-seo.config` is what your client components import, not `seo.ts`.

Beware false positives: grepping for a bare accessor name like `serverStorefront(` also matches the storefront package's own error strings.
