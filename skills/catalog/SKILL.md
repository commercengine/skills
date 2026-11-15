---
name: ce-catalog
description: Commerce Engine product catalog - products, variants, SKUs, categories, faceted search, reviews, and recommendations.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.1.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Products & Catalog

> **Prerequisite**: SDK initialized. See `setup/`. Catalog reads work on the public client (`public()` / `publicStorefront()`) with no anonymous auth; `createProductReview` needs the session client.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| List products | `sdk.catalog.listProducts({ page, limit, category_id })` |
| Get product detail | `sdk.catalog.getProductDetail({ product_id })` |
| List variants | `sdk.catalog.listProductVariants({ product_id })` |
| Get variant detail | `sdk.catalog.getVariantDetail({ product_id, variant_id })` |
| List SKUs (flat) | `sdk.catalog.listSkus()` |
| List categories | `sdk.catalog.listCategories()` |
| Search products | `sdk.catalog.searchProducts({ query: searchTerm, filter?, sort?, facets? })` |
| Get reviews | `sdk.catalog.listProductReviews({ product_id })` |
| Submit review | `sdk.catalog.createProductReview({ product_id }, { ... })` |
| Similar products | `sdk.catalog.listSimilarProducts({ product_id: [productId] })` |
| Upsell products | `sdk.catalog.listUpSellProducts({ product_id: [productId] })` |
| Cross-sell products | `sdk.catalog.listCrossSellProducts({ product_id: [productId] })` |

## Product Hierarchy

Understanding the Product → Variant → SKU relationship is critical:

```
Product (has_variant: false)
  └─ A simple product with one SKU, one price

Product (has_variant: true)
  ├─ Variant A (Color: Red, Size: M) → SKU: "RED-M-001"
  ├─ Variant B (Color: Red, Size: L) → SKU: "RED-L-001"
  └─ Variant C (Color: Blue, Size: M) → SKU: "BLU-M-001"
```

| Concept | Return Type | Description | When to Use |
|---------|-------------|-------------|-------------|
| **Product** | `Product` | Base item with nested `variants` array | PLP where one card per product is desired (e.g., "T-Shirt" card showing color/size selectors) |
| **Variant** | — | A specific option combo (Color + Size) | PDPs, cart items — accessed via `listProductVariants()` or nested in Product |
| **SKU / Item** | `Item` | Flat sellable unit — each variant is its own record | PLP where a flat grid is desired (each color/size combo = separate card), or any page with filters/sorting/search |

## Decision Tree

```
User Request
    │
    ├─ "Show products" / "Product list"
    │   ├─ With filters/sorting/search? → sdk.catalog.searchProducts({ query, filter, sort, facets })
    │   │   → Returns Item[] (flat SKUs) + facet_distribution + facet_stats
    │   ├─ Flat grid (no filters)? → sdk.catalog.listSkus()
    │   │   → Returns Item[] (flat SKUs)
    │   └─ One card per product (group variants)? → sdk.catalog.listProducts()
    │       → Returns Product[] (with nested variants)
    │
    ├─ "Product detail page"
    │   ├─ sdk.catalog.getProductDetail({ product_id })
    │   └─ If has_variant → sdk.catalog.listProductVariants({ product_id })
    │
    ├─ "Search" / "Filter" / "Sort"
    │   └─ sdk.catalog.searchProducts({ query, filter, sort, facets })
    │       → Returns Item[] + facet_distribution + facet_stats
    │
    ├─ "Categories" / "Navigation"
    │   └─ sdk.catalog.listCategories()
    │
    ├─ "Reviews"
    │   ├─ Read → sdk.catalog.listProductReviews({ product_id })
    │   └─ Write → sdk.catalog.createProductReview({ product_id }, body)
    │
    └─ "Recommendations"
        ├─ Similar → sdk.catalog.listSimilarProducts()
        ├─ Upsell → sdk.catalog.listUpSellProducts()
        └─ Cross-sell → sdk.catalog.listCrossSellProducts()
```

## Key Patterns

### Product Listing Page (PLP)

**For PLPs with filters, sorting, or search — use `searchProducts`** (recommended). It returns `Item[]` (flat SKUs) plus `facet_distribution` and `facet_stats` for building filter UI:

```typescript
const { data, error } = await sdk.catalog.searchProducts({
  query: "running shoes",
  filter: "pricing.selling_price 50 TO 200 AND categories.name = footwear",
  sort: ["pricing.selling_price:asc"],
  facets: ["categories.name", "product_type", "tags"],
  page: 1,
  limit: 20,
});

// data.skus → Item[] (flat list — each variant is its own record)
//   product_slug and variant_slug (optional) build SEO-friendly URLs; fall back to product_id
//   item.pricing can be null (no pricing configured) — see "Pricing" below
// data.facet_distribution → { [attribute]: { [value]: count } }
// data.facet_stats → { [attribute]: { min, max } } (e.g. price range)
// data.pagination → { page, limit, total, total_pages }

// filter also accepts arrays — conditions are AND'd:
// filter: ["product_type = physical", "rating >= 4"]
//
// Nested arrays express OR within AND:
// filter: ["pricing.selling_price 50 TO 200", ["categories.name = footwear", "categories.name = apparel"]]
```

**For PLPs without filters** where one card per product is desired (variants grouped under a single card):

```typescript
const { data, error } = await sdk.catalog.listProducts({
  page: 1,
  limit: 20,
  category_id: ["cat_123"],  // Optional: filter by category
});

// data.products → Product[] (each product may contain a variants array)
// Check product.has_variant to know if options exist
// product.pricing is null when has_variant is true (pricing is per variant)
```

**For a flat grid without filters** (each variant = separate card):

```typescript
const { data, error } = await sdk.catalog.listSkus();
// Returns Item[] — same flat type as searchProducts
// product_slug and variant_slug are optional: fall back to product_id for URLs
```

### Product Detail Page (PDP)

```typescript
const { data, error } = await sdk.catalog.getProductDetail({
  product_id: "blue-running-shoes", // Accepts product ID or slug
});

const product = data?.product;

// Prefer product.variants from getProductDetail.
// Fetch separately only if variants are missing or you specifically need the variants endpoint.
if (product?.has_variant && (!product.variants || product.variants.length === 0)) {
  const { data: variantData } = await sdk.catalog.listProductVariants({
    product_id: product.id,
  });
  // variantData.variants contains all options with pricing and stock
}
```

### Pricing

`pricing` is `ProductPricing | null` on `Product`, `ProductDetail`, `Variant`, `VariantDetail` and `Item`:

- `null` on a `Product` / `ProductDetail` with `has_variant: true`: pricing is per variant.
- `null` on any of them when no pricing is configured (for the applicable customer group).

Never read `product.pricing.selling_price` unguarded. For a variant product, read the resolved variant's pricing:

```typescript
// Until the option selection resolves, show the default variant's price
const pricing = product.has_variant
  ? (resolvedVariant ?? product.variants.find((v) => v.is_default))?.pricing ?? null
  : product.pricing;

const price = pricing?.selling_price;  // undefined → render no price (never 0)
const listPrice = pricing?.listing_price;
const currency = pricing?.currency;

// Ordering rules live on pricing too; the API defaults min and step to 1
const minQty = pricing?.min_order_quantity ?? 1;
const step = pricing?.incremental_quantity ?? 1;
const maxQty = pricing?.max_order_quantity ?? null;
```

A selection without pricing has no price to show: keep Add to Cart disabled for it (see `references/pdp-option-model.md`).

### Adding a Product to Cart from PDP

After resolving the variant on the PDP, use `product.id` and `variant.id` from the `getProductDetail` response:

```typescript
// product = data.product from getProductDetail()
// resolvedVariant = the variant matched from product.variants via option selection

if (product.has_variant) {
  // Variant product — must specify both product_id and variant_id
  await sdk.cart.addDeleteCartItem(
    { id: cartId },
    { product_id: product.id, variant_id: resolvedVariant.id, quantity: 1 }
  );
} else {
  // Simple product — variant_id is null
  await sdk.cart.addDeleteCartItem(
    { id: cartId },
    { product_id: product.id, variant_id: null, quantity: 1 }
  );
}
```

`addDeleteCartItem` returns `{ data, error, events }`. `events` carries the server's `Product Added` event with its `event_id`: if the storefront tracks in the browser, pass it to `@commercengine/analytics` so the two copies can deduplicate (see `analytics/`).

With Hosted Checkout, use `addToCart` instead:

```typescript
const { addToCart } = useCheckout();
addToCart(product.id, product.has_variant ? resolvedVariant.id : null, 1);
```

### Canonical Variant URL State (PDP)

Use option query params as source of truth for variant products: `?size=large&color=blue`.

- Query keys should be plain `option.key` values (no custom prefixes).
- Render option groups in `variant_options` order (do not derive option groups by iterating variants).
- Match a variant only when **all** option keys match `variant.associated_options`.
- Normalize option values consistently before comparison:
  - `color` → compare with `option.value.name` (use `option.value.hexcode` for swatch UI)
  - `single-select` → compare with `option.value`
- `variant` query param is optional derived state:
  - Set/update it when options resolve to one variant.
  - Remove it when selection is incomplete/invalid.
  - If URL has only `variant`, backfill option params from that variant.
  - If URL has partial options + valid `variant`, fill only missing options from that variant.
  - If `variant` is invalid, fall back to default (`is_default`, else first variant).
- Disable Add to Cart until required options are selected and the resolved variant is purchasable (`stock_available` or `backorder`) and has `pricing`.
- Disable option values that cannot lead to a purchasable variant under current partial selection.

Use canonical SDK types directly:

```typescript
import type {
  AssociatedOption,
  Product,
  ProductPricing,
  Variant,
  VariantOption,
} from "@commercengine/storefront";
```

Do not derive these app-level types from OpenAPI schemas.

Reference implementation:
- `references/pdp-option-model.md` (URL state + variant matching + attribute/variantOption UI unification + nullable pricing)

### Attribute vs VariantOption (PDP Consistency)

Treat `variant_options` as variant-driving attribute keys (usually `single-select` and `color`), not as a completely separate display domain.

- Non-variant products can still expose `ProductAttribute` values for the same keys.
- Brands may want the same UI treatment for those keys in PDP, even when product has no variants.
- Keep one normalized option-display model:
  - Variant product: selectable options from `variant_options` + `variant.associated_options`.
  - Non-variant product: read-only option-style groups from `attributes` for keys/types the brand wants to style as options.
- If `attribute.key` overlaps a `variant_option.key` on variant products, render the variant-option UI once and avoid duplicate attribute rows.
- Use `attribute.key` alignment plus attribute `type` (`single-select`/`color`) as primary signals.
- Cart rule stays strict: only variant products require variant resolution before Add to Cart.

### Product Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `physical` | Tangible goods requiring shipping | `shipping`, `inventory` |
| `digital` | Downloadable or virtual products | No shipping required |
| `bundle` | Group of products sold together | Contains sub-items |

### Inventory & Availability

- **`stock_available`** — Boolean, **always present** on Product, Variant, and SKU schemas. Use it to disable "Add to Cart" or show "Out of Stock" when `false`.
- **`backorder`** — Boolean, set per product in Admin. When `true`, the product accepts orders even when out of stock. If your business allows backorders, keep the button enabled when `stock_available` is `false` but `backorder` is `true`.
- **Inventory count** — Catalog APIs (`listProducts`, `listSkus`, etc.) support including inventory data in the response. Use this to display numeric stock levels in the UI.

### Customer Groups & Pricing

An advanced feature for B2B storefronts where the admin has configured customer groups (e.g., retailers, stockists, distributors). When `customer_group_id` is sent in API requests, product listings, pricing, and promotions are returned for that specific group.

**Do not pass the header per-call.** Set it once via `defaultHeaders` in SDK config (see `setup/` § "Default Headers"). After the user logs in, update the SDK instance with their group ID — all subsequent SDK calls automatically include it.

### Wishlists

Commerce Engine supports wishlists via `sdk.cart.getWishlist()`, `sdk.cart.addToWishlist({ product_id, variant_id })` and `sdk.cart.removeFromWishlist({ product_id, variant_id })` (the user ID resolves from the session). They return `data.products` as `Item[]`, so the same `pricing` and slug guards apply. `addToWishlist` and `removeFromWishlist` also return `events` (`Product Added to Wishlist` / `Product Removed from Wishlist`) for browser analytics (see `analytics/`). These skills cover the main storefront flows — for wishlists and other secondary features, refer to the [LLM API reference](https://llm-docs.commercengine.io/) or [CE docs](https://www.commercengine.io/docs).

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Building PLP with filters using `listProducts()` | Use `searchProducts({ query, filter, sort, facets })` — it returns `data.skus` (`Item[]`) + `data.facet_distribution` + `data.facet_stats`. `listProducts()` returns `Product[]` with no facets. Uses Meilisearch filter syntax (e.g. `"rating > 4 AND product_type = physical"`). |
| CRITICAL | Confusing `Product` vs `Item` types | `listProducts()` returns `Product[]` (grouped, with variants array). `listSkus()` and `searchProducts()` return `Item[]` (flat — each variant is its own record). |
| CRITICAL | Using `variant` as PDP URL source of truth for multi-option products | Keep option params (`size`, `color`, etc.) canonical; treat `variant` as derived/backfill-only. |
| HIGH | Resolving variants from a single option or `variant_options` only | Match against `variant.associated_options` across **all** option keys. |
| HIGH | Deriving option/variant types from generated schemas in app code | Import SDK types directly from `@commercengine/storefront` (`AssociatedOption`, `VariantOption`, `Variant`, `Product`). |
| MEDIUM | Treating `attributes` and `variant_options` as unrelated PDP UIs | Build a unified option-display model so shared keys (e.g. `metal`) can render consistently across variant and non-variant products. |
| HIGH | Reading `product.pricing.selling_price` (or `item.pricing.x`) unguarded | `pricing` is `null` for products with variants and wherever no pricing is configured: use the resolved variant's pricing and `?.` |
| HIGH | Ignoring `has_variant` flag | Always check `has_variant` before trying to access variant data |
| HIGH | Adding product to cart instead of variant | When `has_variant: true`, must add the specific variant, not the product |
| MEDIUM | Not using `slug` for URLs | Use `slug` field for SEO-friendly URLs — `product_id` params across catalog endpoints accept both IDs and slugs |
| MEDIUM | Assuming `Item.product_slug`, `variant_slug`, `images` or `attributes` are present | They are optional on `Item`: fall back to `product_id` for URLs and `?? []` for lists. Cart lines have no `slug`: use `product_slug` (nullable) |
| MEDIUM | Missing pagination | All list endpoints return `pagination` — use `page` and `limit` params |
| LOW | Re-fetching categories | Categories rarely change — cache them client-side |

## See Also

- `setup/` - SDK initialization required first
- `cart-checkout/` - Adding products to cart
- `orders/` - Products in order context
- `ssr-patterns/` - SSG / pre-rendering for product pages (Next.js `generateStaticParams()`, TanStack Start pre-rendering)

## Documentation

- **Catalog Guide**: https://www.commercengine.io/docs/storefront/catalog
- **API Reference**: https://www.commercengine.io/docs/api-reference/catalog
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/list-products
