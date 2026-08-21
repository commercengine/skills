# Pitfalls

Every entry here is a failure mode seen in a real integration. Each lists the **symptom** first, because the symptom is rarely where the bug is.

## The class of bug that dominates

Almost every defect below is one of two shapes:

1. **Two mechanisms that are individually reasonable and contradict each other** — a hand-rolled tag and a package tag, a static default and a page override, a committed file and a generated one. The fix is always to collapse to one source of truth, not to make both smarter.
2. **Something that typechecks, builds, and lints cleanly while doing nothing** — dead code, an unreachable route, a mistyped result, a wrong hostname. Green CI is not evidence. Inspect the output.

---

## CRITICAL

### Canonical points at a host that does not resolve

**Symptom:** the site works; sitemaps and canonicals reference a domain nobody can reach.

`site.url` feeds canonicals, sitemap entries, `.md` alternates, and breadcrumbs. A wrong host poisons all of them at once, and nothing in a build detects it. Confirm before shipping:

```bash
dig +short acme.example                      # must return records
curl -s -o /dev/null -w "%{http_code}\n" https://acme.example/
```

A value inherited from an existing constants file is not automatically correct — it may only ever have fed `og:url`, where being wrong was cosmetic.

### Duplicate `og:*` tags — crawler reads the wrong one

**Symptom:** `og:title` is the site name on every product page.

No framework deduplicates head content against static markup:

| Framework | Static source | Fix |
|-----------|---------------|-----|
| React SPA | `index.html` | Mark defaults `data-seo-default`, retire the overridden ones on mount |
| Astro | `Layout.astro` | Ternary — package head **or** defaults, never both |
| SvelteKit | `+layout.svelte` | `{#if !page.data.seoHead}` around the defaults |

Count real tags in the output, not substrings:

```bash
grep -o '<meta[^>]*property="og:type"' dist/product/*/index.html | wc -l   # must be 1
```

Tags the package never supplies (`twitter:site`) stay outside the branch.

### Two product schemas on one page

**Symptom:** `["ProductGroup", "BreadcrumbList", "Product"]` in the output.

The package emits `Product`/`ProductGroup`. A hand-rolled product schema left in place describes the same item twice. Delete the hand-rolled one — but keep the breadcrumb, which the package does **not** emit.

Before deleting, check what you are actually removing. `AggregateRating` *is* in the package output, so losing it is not a reason to keep a duplicate; `BreadcrumbList` is not.

### `.md` routes 404 in a static build

**Symptom:** the mirrors work in dev and 404 in production.

Framework routers give a concrete route priority over a catch-all, so `/product/[slug]` claims `/product/shoe.md`. SvelteKit fails the build outright on the collision; the others silently serve HTML. Static builds must use the prebuild script.

### Registration code that never runs

**Symptom:** a feature works on some apps and is silently absent on others; typecheck, build, and lint all pass.

Dead code is valid code. The specific case observed: a registration block placed **after** the cleanup `return` inside `onMount`, so it was unreachable.

```javascript
onMount(() => {
  void bootstrap();
  return () => { cancelled = true; };
  void registerThings();   // ← never runs
});
```

Detect it by looking for a **runtime** signal, not by reading. If a diagnostic callback exists, check the console; if not, add one. A feature with no observable success signal cannot be verified, and that is exactly where this hides.

---

## HIGH

### `BreadcrumbList` silently missing

`productHead` emits the product schema only. `breadcrumbJsonLd` is a separate export the page calls. Easy to overlook precisely because the page still validates.

### Breadcrumb URLs built by concatenation

`${site.url}/product/${slug}` ignores `productBase` and any `routes.product` function. Use `seo.productUrl()` / `seo.categoryUrl()`.

### `getProductDetail` result typed as `Product`

**Symptom:** a type error only when passing the product to the package — *"missing the following properties: description, hsn_code, videos, shipping…"*

`listProducts()` returns `Product`; `getProductDetail()` returns `ProductDetail`, which is wider. A hook declaring the narrow type discards fields the endpoint actually returned. Fix the declaration at its source rather than casting at the call site, and resist patching one field onto the narrow type — widen it properly.

### `listCategories()` without `nested_level`

**Symptom:** *"missing property `child_categories`"* when passing a category to the package.

Pass `{ nested_level: 4 }`. If a helper of your own narrows the type (returning the SDK's exported `Category` alias, which lacks `child_categories`), make the helper generic so it preserves the caller's shape:

```typescript
export function matchCategory<T extends Pick<Category, "id" | "name" | "slug">>(
  categories: T[], slug: string | undefined
): T | undefined
```

### Hook called after an early return

**Symptom:** *"Rendered more hooks than during the previous render"* when data resolves.

`useCommerceSeoHead` must sit with the other hooks, above any `if (isLoading)` / `if (!product)` branch. It renders nothing itself, so its position is free.

### Client bundle pulls in the storefront

Splitting `commerce-seo.config.ts` from `seo.ts` only helps if client components import the former. A single `import { site } from "./seo"` in a client component undoes it, and it still typechecks and builds.

### Duplicate React keys on meta tags

**Symptom:** *"Encountered two children with the same key, `og:image`."*

A product emits one `og:image` per gallery image, so the property name is not unique. Key on property **and** content. React warns that children "may be duplicated and/or omitted" — meaning images can silently vanish.

---

## MEDIUM

### Prebuild fails on CI with `ENOENT: .env.local`

`.env.local` is local-only. Make it optional and fall back to `process.env`. Test the CI path by moving the file aside and exporting the vars — do not assume.

### Env values keep their quotes

`PUBLIC_STORE_ID="abc"` parsed naively yields a store id containing quote characters, producing a 404 that reads as bad credentials. Strip surrounding quotes when parsing an env file.

### Generated files committed before the ignore rule

**Symptom:** `robots.txt` or `settings.json` reappears in unrelated diffs.

`.gitignore` only applies to untracked files. A rule added after the file was committed does nothing. Sweep for the whole class rather than fixing one file:

```bash
git ls-files -i -c --exclude-standard   # tracked files matching an ignore rule
git rm --cached <path>                  # index only; file stays on disk
```

### Blanket `git add -A` in a repo that generates assets

With build output in the working tree, staging everything sweeps up timestamp churn and formatter noise. Stage the files you meant to change.

---

## Verification That Actually Works

| Deployment | How to check the head |
|------------|----------------------|
| Static / prerendered | Read the built HTML directly |
| Next.js | Strip the RSC flight payload first, or serialized metadata reads as duplicate tags |
| TanStack Start | Fetch from a running server; the hydration payload also contains a copy |
| React SPA | A real browser — `curl` sees an empty shell |

Three habits worth keeping:

- **Read source over docs.** The package's own types are authoritative about what it emits.
- **Reproduce before fixing.** A fix for a misdiagnosed cause looks identical to one that works.
- **Prove the test is not vacuous.** A probe that never runs and a build that never bundles the code both report success.

One-per-framework spot checks catch framework-layer bugs and miss per-app ones. The wiring generalises; the app content does not.
