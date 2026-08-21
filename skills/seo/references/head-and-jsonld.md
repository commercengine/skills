# Head Metadata & JSON-LD

How to render `@commercengine/seo` head data in each framework, and the deduplication rules that decide whether a crawler reads your tag or somebody else's.

## The Head Object

`productHead` / `categoryHead` return a plain, framework-neutral shape:

```typescript
interface CommerceSeoHead {
  title: string;
  meta: Array<{ name?: string; property?: string; content: string }>;
  links: Array<{ rel: string; href: string; type?: string }>;
  scripts: Array<{ type: string; content: string }>;  // JSON-LD, pre-serialized
}
```

Two details drive every rendering decision:

- **`property` vs `name`.** Open Graph uses RDFa `property=`; everything else uses `name=`. Branch on which key is present — emitting `name="og:title"` is silently wrong.
- **`scripts[].content` is already escaped** by `serializeJsonLd` (`<`, `>`, `&` → `<` etc.). Do not re-stringify it, and do not escape it again.

## Per-Framework Rendering

### Next.js — use the Metadata API

Next has first-class metadata; do not hand-render tags.

```typescript
// app/product/[slug]/page.tsx
import { createProductMetadata, productOpenGraphTags } from "@commercengine/seo/nextjs";
import { seo } from "@/lib/seo";

export async function generateMetadata({ params }): Promise<Metadata> {
  const { slug } = await params;
  const { data } = await storefront.publicStorefront().catalog.getProductDetail({ product_id: slug });
  if (!data?.product) return { title: slug };
  return createProductMetadata(seo, data.product);
}

export default async function ProductPage({ params }) {
  // ...fetch product...
  const jsonLd = product
    ? await Promise.all([seo.productJsonLd(product), productBreadcrumb(product)])
    : [];

  return (
    <>
      {/* Next's OpenGraphType union has no "product", so these are emitted directly. */}
      {product && productOpenGraphTags(product).map((tag) => (
        <meta key={tag.property} property={tag.property} content={tag.content} />
      ))}
      {jsonLd.map((schema) => (
        <script
          key={schema["@type"]}
          type="application/ld+json"
          dangerouslySetInnerHTML={{ __html: safeJsonLd(schema) }}
        />
      ))}
      {/* ...page... */}
    </>
  );
}
```

> `metadata.other` emits `name=`, not `property=`. That is why product-namespace OG tags come from `productOpenGraphTags` instead.

For categories use `createCategoryMetadata(seo, category)`. It needs a real `Category` object — do **not** title-case the URL slug as a stand-in, or the name, description, and canonical will all be invented.

### Astro — a `seoHead` prop on the layout

```astro
---
// src/pages/product/[slug].astro
import { breadcrumbJsonLd } from "@commercengine/seo";
import { createAstroProductHead } from "@commercengine/seo/astro";
import Layout from "../../layouts/Layout.astro";
import { seo, site } from "../../lib/seo";

const product = /* ...fetch... */;
const seoHead = product ? await createAstroProductHead(seo, product) : undefined;

// `productHead` emits the product schema only — the breadcrumb stays the page's job.
const origin = site.url.replace(/\/+$/, "");
const category = product?.categories?.[0];
const breadcrumb = breadcrumbJsonLd([
  { name: "Home", url: `${origin}/` },
  ...(category?.name && category?.slug
    ? [{ name: category.name, url: (await seo.categoryUrl(category)) ?? origin }]
    : []),
  { name: product?.name ?? "Product", url: (await seo.productUrl(product)) ?? origin },
]);
---
<Layout seoHead={seoHead} title={fallbackTitle}>
  <script is:inline slot="head" type="application/ld+json" set:html={JSON.stringify(breadcrumb)} />
  <ProductContent client:idle serverProduct={product} />
</Layout>
```

```astro
---
// src/layouts/Layout.astro
import type { CommerceSeoHead } from "@commercengine/seo";
interface Props { seoHead?: CommerceSeoHead; title?: string; /* ... */ }
const { seoHead, title } = Astro.props;
---
<head>
  {seoHead ? (
    <>
      <title>{seoHead.title}</title>
      {seoHead.meta.map((tag) => <meta {...tag} />)}
      {seoHead.links.map((tag) => <link {...tag} />)}
      {seoHead.scripts.map((tag) => <script is:inline type={tag.type} set:html={tag.content} />)}
    </>
  ) : (
    <>
      {/* sitewide defaults — the ELSE branch, never alongside */}
      <title>{title}</title>
      <meta property="og:title" content={title} />
      {/* ... */}
    </>
  )}
  {/* Tags the package never supplies stay outside the branch. */}
  <meta name="twitter:site" content="@acme" />
  <slot name="head" />
</head>
```

**The ternary is the point.** Astro does not deduplicate head content. Rendering the defaults *and* the package head puts two `og:title` tags on the page.

### SvelteKit — build in `+page.server.ts`, render in `<svelte:head>`

Head building is async and the resolvers live server-side, so do it in the load function.

```typescript
// src/routes/product/[slug]/+page.server.ts
import { createSvelteKitProductHead, serializeJsonLd } from "@commercengine/seo/sveltekit";
import { seo } from "$lib/commerce-seo";

export const load: PageServerLoad = async ({ params }) => {
  // ...fetch product, 404 on genuine absence...
  const [seoHead, breadcrumb] = await Promise.all([
    createSvelteKitProductHead(seo, product),
    productBreadcrumb(product),   // returns serializeJsonLd(...) — a string
  ]);
  return { product, seoHead, breadcrumb };
};
```

```svelte
<svelte:head>
  {#if data.seoHead}
    <title>{data.seoHead.title}</title>
    {#each data.seoHead.meta as tag}
      {#if tag.property}
        <meta property={tag.property} content={tag.content} />
      {:else}
        <meta name={tag.name} content={tag.content} />
      {/if}
    {/each}
    {#each data.seoHead.links as link}
      <link rel={link.rel} href={link.href} type={link.type} />
    {/each}
    {#each data.seoHead.scripts as script}
      {@html `<script type="${script.type}">${script.content}</script>`}
    {/each}
    {@html `<script type="application/ld+json">${data.breadcrumb}</script>`}
  {:else}
    <!-- fallback for the resilient no-product render -->
  {/if}
</svelte:head>
```

The root `+layout.svelte` must yield its defaults when a page supplies head data — SvelteKit renders **both** `<svelte:head>` blocks otherwise:

```svelte
<svelte:head>
  {#if !page.data.seoHead}
    <meta property="og:site_name" content={SITE_NAME} />
    <meta property="og:image" content={OG_IMAGE} />
  {/if}
  <meta name="twitter:site" content={TWITTER_SITE} />
</svelte:head>
```

### TanStack Start — build in the loader, `head` is synchronous

```typescript
export const Route = createFileRoute("/product/$slug")({
  loader: async ({ params }) => {
    const product = /* ...fetch... */;
    // `head` cannot await, so head data is built here and passed through loaderData.
    return { product, seoHead: product ? await productHeadWithBreadcrumb(product) : undefined };
  },
  head: ({ loaderData }) => loaderData?.seoHead ?? {},
  component: ProductPage,
});
```

`createTanStackStartProductHead` returns TanStack's shape directly — `{ meta, links, scripts }`, where `scripts` entries use `children`, not `content`. To append a breadcrumb:

```typescript
return {
  ...head,
  scripts: [...head.scripts, { type: "application/ld+json", children: serializeJsonLd(breadcrumb) }],
};
```

`<HeadContent />` must be present in `__root.tsx` or nothing renders.

### React SPA — a `<Seo>` component fed by a hook

Route resolution is async, so head data cannot be computed inline during render.

```tsx
export function useCommerceSeoHead(
  entity: ProductDetail | Category | null | undefined,
  kind: "product" | "category",
): CommerceSeoHead | null {
  const [head, setHead] = useState<CommerceSeoHead | null>(null);
  useEffect(() => {
    if (!entity) { setHead(null); return; }
    let active = true;
    const build = kind === "product"
      ? seo.productHead(entity as ProductDetail).then((h) => withBreadcrumb(h, entity as ProductDetail))
      : seo.categoryHead(entity as Category);
    void build.then((v) => { if (active) setHead(v); }).catch(() => { if (active) setHead(null); });
    return () => { active = false; };
  }, [entity, kind]);
  return head;
}
```

```tsx
export function Seo({ head }: { head: CommerceSeoHead | null }) {
  if (!head) return null;
  return (
    <>
      <title>{head.title}</title>
      {/* A product emits one og:image per gallery image, so the NAME is not a unique key. */}
      {head.meta.map((tag) =>
        tag.property
          ? <meta key={`${tag.property}:${tag.content}`} property={tag.property} content={tag.content} />
          : <meta key={`${tag.name}:${tag.content}`} name={tag.name} content={tag.content} />
      )}
      {head.links.map((tag) => (
        <link key={`${tag.rel}:${tag.href}`} rel={tag.rel} href={tag.href} type={tag.type} />
      ))}
      {head.scripts.map((s) => (
        <script key={s.content.slice(0, 64)} type={s.type}
                dangerouslySetInnerHTML={{ __html: s.content }} />
      ))}
    </>
  );
}
```

**Call the hook above every early return.** Placing it after an `if (isLoading)` or `if (!product)` branch changes the hook count between renders, and React throws *"Rendered more hooks than during the previous render"* the moment the product resolves.

## Retiring Static Defaults (SPA)

React 19 hoists `<title>`/`<meta>`/`<link>` into `<head>` but does **not** deduplicate them against static markup in `index.html`. If your `index.html` carries sitewide OG tags, a product page ships two `og:title` and two `og:type`, and crawlers read the first — the site name, not the product.

Mark the defaults and retire the ones a page overrides:

```html
<meta data-seo-default property="og:title" content="Acme" />
<meta data-seo-default property="og:image" content="https://acme.example/og.jpg" />
<meta data-seo-default property="og:image:width" content="1200" />
```

```tsx
function useRetireDefaults(head: CommerceSeoHead | null) {
  useEffect(() => {
    if (!head) return;
    const supplied = new Set(head.meta.map((t) => t.property ?? t.name ?? ""));
    const removed: Array<{ node: Element; parent: Node; next: Node | null }> = [];
    for (const node of document.head.querySelectorAll("meta[data-seo-default]")) {
      const key = node.getAttribute("property") ?? node.getAttribute("name") ?? "";
      // `og:image:width` belongs to the default `og:image`; drop it with its parent tag.
      const overridden = supplied.has(key) || [...supplied].some((e) => key.startsWith(`${e}:`));
      if (!overridden) continue;
      removed.push({ node, parent: node.parentNode as Node, next: node.nextSibling });
      node.remove();
    }
    // Restore on unmount so pages without head data keep the defaults after a client-side nav.
    return () => { for (const { node, parent, next } of removed) parent.insertBefore(node, next); };
  }, [head]);
}
```

## JSON-LD

### What the package emits

| Product shape | Schema |
|---------------|--------|
| `has_variant: false` | `Product` with `offers: Offer` |
| `has_variant: true` | `ProductGroup` with `hasVariant[]` and `variesBy[]` |

`AggregateRating` is included automatically when `reviews_count > 0` — do not hand-roll it.

### `variesBy`

Google supports six variant dimensions. The package maps a `variantOption` to a schema.org URI only when it can do so unambiguously:

- `type: "color"` is identified **structurally** (its value is an object, not a string), so a colour axis is always recognised regardless of what it is called.
- `size`, `material`, `pattern`, `suggestedGender` are matched by normalised exact name (case, spaces, `_` and `-` are ignored).
- Anything else stays a plain `Text` axis on `hasVariant` and is not claimed as a Google dimension.

`suggestedAge` is deliberately **not** supported: schema.org types it as a `QuantitativeValue`, and inferring a numeric range from arbitrary merchant text would be guessing.

Attributes that repeat a variant axis are excluded from `additionalProperty`, so a value never appears twice.

### Breadcrumbs are yours

`productHead` emits the product schema only. Build the trail with the resolvers so a custom route base stays correct:

```typescript
async function productBreadcrumb(product: ProductDetail) {
  const category = product.categories?.[0];
  const home = seo.config.site.url;
  const [productUrl, categoryUrl] = await Promise.all([
    seo.productUrl(product),
    category ? seo.categoryUrl(category) : Promise.resolve(null),
  ]);
  return seo.breadcrumbJsonLd([
    { name: "Home", url: home },
    ...(category && categoryUrl ? [{ name: category.name, url: categoryUrl }] : []),
    { name: product.name, url: productUrl ?? home },
  ]);
}
```

## Verifying

Never trust a green build. Inspect the **output**:

```bash
# Static/prerendered builds — count real tags, not substrings
grep -o '<meta[^>]*property="og:type"' dist/product/*/index.html | wc -l   # must be 1
grep -o '<link[^>]*rel="canonical"' build/product/*.html | wc -l           # must be 1
```

- **Next.js**: strip the RSC flight payload first (`<script>self.__next_f.push(...)</script>`) or serialized metadata reads as duplicate tags.
- **TanStack Start**: the hydration payload also contains a copy — match `<link ...>` tags, not the bare string.
- **SPA**: curl shows nothing; the head is client-rendered. Use a real browser and read the DOM after hydration.
