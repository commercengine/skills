---
name: ce-catalog
description: Commerce Engine product catalog - products, variants, SKUs, categories, faceted search, reviews, and recommendations.
license: MIT
allowed-tools: WebFetch
metadata:
  author: commercengine
  version: "1.0.0"
---

# Products & Catalog

> **Prerequisite**: SDK initialized and anonymous auth completed. See `setup/`.

## Quick Reference

| Task | SDK Method |
|------|-----------|
| List products | `sdk.catalog.listProducts({ query: { page, limit, category_id } })` |
| Get product detail | `sdk.catalog.getProduct({ product_id_or_slug })` |
| List variants | `sdk.catalog.getProductVariants({ product_id })` |
| Get variant detail | `sdk.catalog.getProductVariantDetail({ product_id, variant_id })` |
| List SKUs (flat) | `sdk.catalog.listSkus()` |
| List categories | `sdk.catalog.listCategories()` |
| Search products | `sdk.catalog.searchProducts({ query: { q: searchTerm } })` |
| Get reviews | `sdk.catalog.getProductReviews({ product_id })` |
| Submit review | `sdk.catalog.createProductReview({ product_id }, { ... })` |
| Similar products | `sdk.catalog.getSimilarProducts({ query: { product_id } })` |
| Upsell products | `sdk.catalog.getUpsellProducts({ query: { product_id } })` |
| Cross-sell products | `sdk.catalog.getCrossSellProducts({ query: { product_id } })` |

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

| Concept | Description | When to Use |
|---------|-------------|-------------|
| **Product** | The base item (e.g., "T-Shirt") | Product listing pages (PLPs) |
| **Variant** | A specific option combo (Color + Size) | Product detail pages (PDPs), cart items |
| **SKU** | Flat sellable unit with unique identifier | Inventory checks, direct purchase |

## Decision Tree

```
User Request
    │
    ├─ "Show products" / "Product list"
    │   ├─ Need flat list? → sdk.catalog.listSkus()
    │   └─ Need hierarchy? → sdk.catalog.listProducts()
    │
    ├─ "Product detail page"
    │   ├─ sdk.catalog.getProduct({ product_id_or_slug })
    │   └─ If has_variant → sdk.catalog.getProductVariants({ product_id })
    │
    ├─ "Search"
    │   └─ sdk.catalog.searchProducts({ query: { q } })
    │       → Returns items + facets for filtering
    │
    ├─ "Categories" / "Navigation"
    │   └─ sdk.catalog.listCategories()
    │
    ├─ "Reviews"
    │   ├─ Read → sdk.catalog.getProductReviews({ product_id })
    │   └─ Write → sdk.catalog.createProductReview({ product_id }, body)
    │
    └─ "Recommendations"
        ├─ Similar → sdk.catalog.getSimilarProducts()
        ├─ Upsell → sdk.catalog.getUpsellProducts()
        └─ Cross-sell → sdk.catalog.getCrossSellProducts()
```

## Key Patterns

### Product Listing Page (PLP)

```typescript
const { data, error } = await sdk.catalog.listProducts({
  query: {
    page: 1,
    limit: 20,
    category_id: "cat_123",  // Optional: filter by category
  },
});

if (data) {
  data.products.forEach((product) => {
    console.log(product.name, product.selling_price, product.slug);
    // Check product.has_variant to know if options exist
  });
}
```

### Product Detail Page (PDP)

```typescript
// Get product by slug or ID
const { data, error } = await sdk.catalog.getProduct({
  product_id_or_slug: "blue-running-shoes",
});

const product = data?.product;

// If product has variants, fetch them
if (product?.has_variant) {
  const { data: variantData } = await sdk.catalog.getProductVariants({
    product_id: product.id,
  });
  // variantData.variants contains all options with pricing and stock
}
```

### Faceted Search

```typescript
const { data, error } = await sdk.catalog.searchProducts({
  query: { q: "running shoes" },
});

// data.items → matching products (flat SKU-level)
// data.facets → attribute filters (brand, color, size, price range)
```

### Product Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `physical` | Tangible goods requiring shipping | `shipping`, `inventory` |
| `digital` | Downloadable or virtual products | No shipping required |
| `bundle` | Group of products sold together | Contains sub-items |

### Customer Groups & Pricing

Pass `x-customer-group-id` header for group-specific pricing:

```typescript
// SDK handles this via customer group context
// Different customer groups see different prices, promotions, and subscription rates
```

## Common Pitfalls

| Level | Issue | Solution |
|-------|-------|----------|
| CRITICAL | Using `listProducts()` when you need flat items | Use `listSkus()` for flat purchasable units, `listProducts()` for hierarchy |
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
- **LLM Reference**: https://llm-docs.commercengine.io/storefront/operations/list-all-products
