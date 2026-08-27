# CMS-Owned URLs: Route Records

Read this when the storefront's public URLs come from a CMS rather than from the Commerce Engine
catalog. If `/products/<ce-slug>` *is* the real URL, you do not need any of it — set
`routes.productBase` and stop.

## The fork

```
Who decides a product's public URL?
    │
    ├─ THE CATALOG   → routes: { productBase, categoryBase }
    │                  The CE slug is the URL. Nothing else to declare.
    │
    └─ A CMS         → routes: createCmsRoutes({ source })
                       Pages declare themselves. Several pages may sell one product.
```

The second case is not "the first case with a function". A resolver answers *"what is this product's
URL?"*, and a CMS-fronted storefront has three different questions:

| Question | Surface | Answered by |
|---|---|---|
| Which page is being rendered? | PDP head, `.md` mirror, metadata | the **hint** you pass |
| Which single page do I link to? | product cards, category rows, agent results | the **primary** record |
| What are *all* the public pages? | `sitemap.xml`, `sitemap.md`, static generation | **every** record |

A resolver returning one string can only answer the second. Where six landing pages sell one product,
the other five never reach a sitemap.

## A record

A record inverts the resolver: instead of deriving a URL from an entity, it *states* that a path
exists and names the entity behind it.

```typescript
import type { CommerceRouteRecord } from "@commercengine/seo/routes";

const records: CommerceRouteRecord[] = [
  { kind: "product",  path: "/products/rub-on-relief",      productId: "01H…", productSlug: "easy-to-rub-emulsion" },
  { kind: "product",  path: "/products/easy-to-rub-emulsion", productId: "01H…", productSlug: "easy-to-rub-emulsion" },
  { kind: "product",  path: "/products/discontinued-amp",   productId: "01J…", markdown: false },
  { kind: "category", path: "/category/audio",              categoryId: "01K…", categorySlug: "audio" },
];
```

| Field | Meaning |
|---|---|
| `path` | The public path. Leading slash enforced, trailing stripped, percent-encoding normalized. |
| `productId` / `categoryId` | The CE entity. A category may be identified by `categoryId`, `categorySlug`, or both — the type makes "neither" unrepresentable. |
| `productSlug` / `categorySlug` | The **catalog** slug. Optional, but it is what lets the manifest pick a primary without being told. |
| `primary` | Override the automatic choice. Only meaningful when an entity has several routes. Two on one entity is a construction error. |
| `markdown: false` | This page exists but its Markdown mirror does not. See below. |
| `lastModified` | Per **page**, for `sitemap.xml`. Six landing pages were edited on six different days. |

### What construction rejects

Rejections are loud on purpose — each one is a URL that would resolve outbound and 404 inbound:

| Input | Why it is refused |
|---|---|
| `/products/x?variant=b`, `/products/x#tab` | A record names a page; variants are applied by `variantPath` |
| `/products/a/b` (nested) | The request matcher is `^<base>/([^/]+?)(?:\.md)?$` and framework rewrites are `:slug` |
| `/shop/x` while `productBase` is `/products` | Outside the base every matcher keys off |
| Two records claiming one path | One path cannot mean two things |
| `/products/x.md` | The reserved mirror suffix — the page and its mirror need distinct paths |
| `..` or `%2E%2E` as a segment | URL parsing resolves them away, so the emitted URL and the matcher would disagree |
| A lone `%` or other malformed escape | It cannot be decoded into a key |
| Blank or whitespace-only identifiers | They become map keys |

**Route bases are checked more strictly than record paths**, and the two are easy to conflate. A base
additionally rejects a backslash, an encoded separator (`%2F`, `%5C`) and a network-path reference
(`//host`, `/\host`), because a base is concatenated into matchers and rewrites. A record path accepts
a raw or encoded backslash and emits it as `%5C` — `/products/a\b` becomes `/products/a%5Cb` — but
still rejects `%2F`, because decoding it creates a nested route that the matcher cannot serve.

## Wiring it: `createCmsRoutes`

This is the whole server-side integration. It returns a plain `CommerceSeoRoutes` — spread it and
override any hook.

```typescript
// lib/seo.ts — SERVER-BOUND
import { createCommerceSeo } from "@commercengine/seo";
import { createCmsRoutes } from "@commercengine/seo/routes";
import { commerceSeo } from "./commerce-seo.config";
import { serverStorefront } from "./storefront";

export const seo = createCommerceSeo({
  ...commerceSeo,
  storefront: serverStorefront,
  routes: createCmsRoutes({
    source: async () => buildRecordsFromCms(),   // your CMS read
    ttlMs: 60_000,                               // default; 0 disables caching
  }),
});
```

It fills every hook from one cached manifest: `product`, `category`, `productRoutes`,
`categoryRoutes`, `categoryMarkdownRoute`, `resolveProductRoute`, `resolveCategoryRoute`, and `list`.

**The TTL is not an optimisation.** `routes.*` is called once per entity on catalog-wide passes, so a
CMS read per product makes `/sitemap.md` quadratic. It is a TTL rather than a permanent memo because
page publishes have to become visible without a redeploy, and framework cache primitives do not exist
in every runtime this runs in (middleware, the browser).

## Which page a link goes to

Three tiers, applied in order:

| | Rule | When it applies |
|---|---|---|
| 1 | a record marked `primary: true` | you deliberately override the convention |
| 2 | the record whose segment equals the catalog slug | the normal case, needs `productSlug` |
| 3 | **`null`** — no destination, reported by `conflicts()` | nothing named a page |

Rule 3 is deliberately not "pick one". An entity nobody chose a page for has no destination a link
can honestly name, so callers render a **non-link** rather than sending a shopper somewhere chosen by
`sort()`. Pass `ambiguous: "first"` for the lexically first route instead — a working link to a real
page for the right entity, at the cost of the destination being incidental.

Enumeration ignores this entirely: an ambiguous entity keeps every one of its pages in sitemaps and
static generation. Only the *link* goes dark.

```typescript
for (const { kind, id, paths } of manifest.conflicts()) {
  console.warn(`[routes] ${kind} ${id} has ${paths.length} pages and the catalog names none:`, paths);
}
```

**`conflicts()` cannot see one case.** It is computed at construction from the records alone. Where
one record identifies a category only by ID and another only by slug, nothing in the records says
they are the same category — the catalog object supplies that at lookup time, and only then can the
combined group prove to have no primary. Gate on resolution too, which also catches entities with no
page at all:

```typescript
const undecided = categories.filter((category) => !manifest.categoryPath(category));
if (undecided.length) throw new Error(`[routes] no linkable page: ${undecided.map((c) => c.slug).join(", ")}`);
```

## Links in the browser

`CommerceSeoRoutes.product` may be async, so no client component can write `href={...}`. For client
code, build a manifest — every method is **synchronous**:

```typescript
import { createCommerceRouteManifest } from "@commercengine/seo/routes";

const manifest = createCommerceRouteManifest(await loadRecords());

<a href={manifest.productPath(item) ?? undefined}>{item.product_name}</a>
```

It accepts the real catalog types with no field remapping — a listing `Item` (`product_id`,
`product_slug`, `variant_slug`), a `Product`/`ProductDetail` (`id`, `slug`), a `ProductRouteInput`, or
a bare ID string. A `Category` is a type error and resolves to `null` at runtime, so it cannot be
mistaken for a product.

| Method | Answers |
|---|---|
| `productPath(ref, { variantSlug })` | the primary path, or `null` |
| `categoryPath(ref)` | same for a category; `ref` may be a string, `{ id }`, `{ slug }`, or the catalog object |
| `productRoutes(ref)` / `categoryRoutes(ref)` | every route, primary first |
| `resolve(path)` | full path → record. Full paths only — `wellness` may be both a product and a category |
| `resolveProduct(pathOrSegment)` / `resolveCategory(…)` | when the kind is known |
| `list(filter?)`, `toJSON()` | the whole space, ready to serialize |
| `conflicts()` | entities nothing settled |

**Do not ship a 100,000-product manifest to resolve one search result.** The package performs no
route-data transport and imposes no API, batching or cache policy. At that size, resolve the handful
of visible items server-side and pass their hrefs down, or back `routes.product` with a
storefront-owned point cache. A small route table in the RSC payload is fine; a whole catalog is not.

**Never invent a fallback when `productPath` returns `null`.** `null` means the storefront has no
known public page, and `/products/${ce_slug}` is precisely the URL that 404s — it is the bug this
model exists to fix.

## Rendering the right page

A product with six landing pages renders six times, and each render must know *which* page it is.
That is the hint — a trailing optional argument on every render method:

```typescript
// app/products/[slug]/page.tsx
const record = await cms.pageBySlug(slug);          // the render already fetches this

const page = await seo.productPage(product, {
  path: `/products/${slug}`,
  slug,
  markdown: record.hasCeProduct,                    // see below
});
page.head();      // canonical, JSON-LD, .md alternate — all for THIS page
page.markdown();  // this page's mirror
```

Without a hint the package answers with the **primary**, which is right for a link and wrong for a
render: five of the six pages would declare the sixth as their canonical.

`generateMetadata` and the page body are separate invocations, so both need the hint. Wrap the CMS
read in React `cache()` so they share one fetch.

## `markdown: false` — the page exists, the mirror does not

Two separate claims. A `.md` document is rendered from the catalog entity, so a page whose product
has been delisted still serves HTML and still belongs in `sitemap.xml`, while `/that-page.md` has
nothing to render.

Everything that could advertise a mirror reads this rather than assuming:

| Surface | With `markdown: false` | Reads it from |
|---|---|---|
| `/sitemap.xml` | still listed — it is a real page | the record |
| Static generation | writes no `.md` file for it | the record |
| Request handler `/that-page.md` | `404`, without touching the catalog | the record |
| `/llms.txt` | omits the category, or links its first renderable route | the route adapter |
| `productHead` / `createProductMetadata` | no `<link rel="alternate" type="text/markdown">` | **the hint** |
| `productMarkdown` frontmatter | no `markdown_url` | **the hint** |

**`/sitemap.md` differs by deployment mode, so check which one you are shipping.** Verified against
the published package with one `markdown: false` product:

| Deployment | `/sitemap.md` entry |
|---|---|
| Runtime handler | `- [Discontinued Amp](https://acme.example/products/discontinued-amp)` — the **HTML** page, no `.md` |
| Static generation | **absent** — the index is built from the assets that were emitted, and no file was |

Both are safe, in that neither advertises a mirror that 404s. They differ in whether an agent reading
`sitemap.md` learns the page exists at all. If that matters to you, keep the page discoverable
through `sitemap.xml`, which lists it in both modes.

The hint column matters as much as the surface column. The first four rows run where the whole route
space is already in hand. The last three run while rendering one page, where it is not — so the fact
travels on the hint rather than being looked up. Consulting the route space during a render would
load and index every route the storefront has, in every cold serverless isolate, to produce one
boolean.

`seo.productMarkdownAvailable(slug)` answers the same question from the declared space and suits a
long-lived server or a build. It calls `routes.list()`, so do not reach for it per render.

## Inbound: public slug → CE identifier

Outbound alone is half the job. `/products/rub-on-relief.md` is looked up in the catalog by the
literal string `rub-on-relief` and 404s unless something maps it back:

```typescript
resolveProductRoute: async (publicSlug) => manifest.resolveProduct(publicSlug)?.productId ?? publicSlug,
```

`createCmsRoutes` wires this for you. The fallback to `publicSlug` keeps a page published since the
last cache refresh working whenever the CMS slug and the catalog slug happen to agree.

This is also what stops an agent that reads a CMS slug off the page and calls
`get_product("rub-on-relief")` from hitting a catalog that has never heard of it.

## Enumeration reaches discovery

With `routes.list` configured, the declared route space **replaces** the catalog walk as the list of
pages. It cannot replace the catalog as the source of what to render *with* — each document still
needs its entity, fetched by the identifier its record carries.

Consequences worth knowing:

- Sitemap shards pack against real URLs, so a catalog whose pages expand 6:1 cannot advertise one
  shard and then trip the 50,000-URL assertion.
- Static generation renders **each route** with its own hint, its own canonical and its own
  `enrichProduct` result. Six sitemap entries backed by one `.md` file would publish five broken
  mirrors.
- A CMS page for a delisted or inactive product is reachable at all, which a catalog walk could never
  make it.

`seo.routeSpace()` returns the space as absolute URLs; `seo.routeManifest()` returns the manifest
itself. Both are `null` when `routes.list` is not configured — deliberately distinct from an empty
array, which means "this storefront genuinely has no routes".

## Serving records to the browser

**Only where the whole route space is small enough to be a page asset.** A few hundred records is a
sensible payload; a hundred thousand is a catalog download on every cold client, and no amount of
caching makes that the right shape. Decide by measuring the serialized size, not by feel:

```bash
curl -s https://acme.example/api/commerce-routes | wc -c   # treat 100 KB as the line to justify
```

Under that line, serve the whole space and let clients resolve synchronously:

```typescript
// app/api/commerce-routes/route.ts — SMALL route spaces only
export const revalidate = 60;
export async function GET() {
  return Response.json((await seo.routeManifest())?.toJSON() ?? []);
}
```

`toJSON()` emits normalized records that `createCommerceRouteManifest` re-indexes identically, so the
client and the server resolve the same URLs.

Over that line, do not ship the space at all. Two shapes that scale instead:

| Situation | Shape |
|---|---|
| Server-rendered cards and rows | Resolve `href` on the server for the items on the page and pass strings to the client. Nothing about routing crosses the boundary. |
| Client-only surfaces (live search) | A point-lookup endpoint keyed by the IDs in the result set, backed by a storefront-owned cache. Resolve the handful being displayed. |

Both keep one URL space. The rule that must not bend is the fallback rule: when resolution yields
nothing, render a non-link. A guessed `/products/${ce_slug}` is the 404 this model exists to remove.

## Visual-editing CMSs

DatoCMS, Sanity and Contentful embed invisible Unicode (stega) in every string in draft mode. A slug
that looks identical is a different string to `===` and a different key in every index built from it,
so **every lookup misses and every URL becomes `null`**. Strip it where you build the records:

```typescript
import { vercelStegaClean } from "@vercel/stega";
const slug = vercelStegaClean(page.slug);
```

## Verify it

```bash
# Every mirror sitemap.md advertises must actually render.
curl -s https://acme.example/sitemap.md \
  | grep -oE 'https?://[^)]+\.md' \
  | while read -r url; do
      code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
      [ "$code" = "200" ] || echo "MISSING $code $url"
    done

# How many records the browser is actually given.
curl -s https://acme.example/api/commerce-routes | jq 'length'
```

The check that matters most is the one that compares two surfaces: for every page in `sitemap.xml`,
does the HTML advertise a `.md` alternate *if and only if* that `.md` returns 200? A mismatch there is
the failure this whole model exists to prevent.
