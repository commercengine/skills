---
name: ce-catalog
description: Commerce Engine product catalog - products, variants, SKUs, categories, faceted search, reviews, and recommendations.
license: MIT
allowed-tools: Bash
metadata:
  author: commercengine
  version: "1.0.0"
---

> **LLM Docs Header**: All requests to `https://llm-docs.commercengine.io` **must** include the `Accept: text/markdown` header (or append `.md` to the URL path). Without it, responses return HTML instead of parseable markdown.

# Products & Catalog

> **Prerequisite**: SDK initialized and anonymous auth completed. See `setup/`.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| List products | `sdk.catalog.listProducts({ page, limit, category_id })` |
| Get product detail | `sdk.catalog.getProductDetail({ product_id_or_slug })` |
| List variants | `sdk.catalog.listProductVariants({ product_id })` |
| Get variant detail | `sdk.catalog.getVariantDetail({ product_id, variant_id })` |
| List SKUs (flat) | `sdk.catalog.listSkus()` |
| List categories | `sdk.catalog.listCategories()` |
| Search products | `sdk.catalog.searchProducts({ query: searchTerm, filter?, sort?, facets? })` |
| Get reviews | `sdk.catalog.listProductReviews({ product_id })` |
| Submit review | `sdk.catalog.createProductReview({ product_id }, { ... })` |
| Similar products | `sdk.catalog.listSimilarProducts({ product_id })` |
| Upsell products | `sdk.catalog.listUpSellProducts({ product_id })` |
| Cross-sell products | `sdk.catalog.listCrossSellProducts({ product_id })` |

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
    │   ├─ sdk.catalog.getProductDetail({ product_id_or_slug })
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
```

**For a flat grid without filters** (each variant = separate card):

```typescript
const { data, error } = await sdk.catalog.listSkus();
// Returns Item[] — same flat type as searchProducts
```

### Product Detail Page (PDP)

```typescript
const { data, error } = await sdk.catalog.getProductDetail({
  product_id_or_slug: "blue-running-shoes",
});

const product = data?.product;

// If product has variants, fetch them
if (product?.has_variant) {
  const { data: variantData } = await sdk.catalog.listProductVariants({
    product_id: product.id,
  });
  // variantData.variants contains all options with pricing and stock
}
```

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

Commerce Engine supports wishlists (add, remove, fetch) via SDK methods. These skills cover the main storefront flows — for wishlists and other secondary features, refer to the [LLM API reference](https://llm-docs.commercengine.io/) or [CE docs](https://www.commercengine.io/docs).

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Building PLP with filters using `listProducts()` | Use `searchProducts({ query, filter, sort, facets })` — it returns `data.skus` (`Item[]`) + `data.facet_distribution` + `data.facet_stats`. `listProducts()` returns `Product[]` with no facets. Uses Meilisearch filter syntax (e.g. `"rating > 4 AND product_type = physical"`). |
| CRITICAL | Confusing `Product` vs `Item` types | `listProducts()` returns `Product[]` (grouped, with variants array). `listSkus()` and `searchProducts()` return `Item[]` (flat — each variant is its own record). |
| HIGH | Ignoring `has_variant` flag | Always check `has_variant` before trying to access variant data |
| HIGH | Adding product to cart instead of variant | When `has_variant: true`, must add the specific variant, not the product |
| MEDIUM | Not using `slug` for URLs | Use `slug` field for SEO-friendly URLs, `product_id_or_slug` accepts both |
| MEDIUM | Missing pagination | All list endpoints return `pagination` — use `page` and `limit` params |
| LOW | Re-fetching categories | Categories rarely change — cache them client-side |

## See Also

- `setup/` - SDK initialization required first
- `cart-checkout/` - Adding products to cart
- `orders/` - Products in order context
- `nextjs-patterns/` - SSG for product pages with `generateStaticParams()`

## Documentation

- **Catalog Guide**: https://www.commercengine.io/docs/storefront/catalog
- **API Reference**: https://www.commercengine.io/docs/api-reference/catalog
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/list-products
