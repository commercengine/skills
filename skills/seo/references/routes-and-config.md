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

Defaults are `/product` and `/category`. `search` defaults to `/search?q=`.

### Custom: a function per entity

```typescript
routes: {
  product: (input) => `/shop/${input.categorySlug ?? "all"}/${input.slug}`,
  category: (category) => `/collections/${category.slug}`,
}
```

Return `null` to **omit** an entity entirely. Omission is honoured everywhere at once — sitemap, `llms.txt`, category listings, and the `.md` handler (which then 404s). Use it for unpublished or gated products rather than filtering in three places.

Both may be async, so a CMS lookup is fine.

### Inbound: mapping public slugs back to CE

If your public URLs use CMS slugs rather than Commerce Engine slugs, the outbound function alone is not enough — the `.md` handler needs to resolve an incoming public slug back to something the catalog understands:

```typescript
routes: {
  product: (input) => `/products/${cmsSlugFor(input.id)}`,
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

```typescript
indexable: true       // force index
indexable: false      // force noindex
// omitted           → detectDeploymentEnvironment()
```

Detection reads the platform environment (Vercel, Netlify, and friends) and treats preview deployments as non-production. Two consequences worth internalising:

- **Unset means `noindex`.** Safe by default; a live site needs the flag set.
- **A local build is not a production deployment.** The prebuild script should pass `indexable: true` explicitly, or every generated `robots.txt` will disallow everything.

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
