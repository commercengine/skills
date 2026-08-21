# Deployment Modes

Where `robots.txt`, `sitemap.xml`, `llms.txt`, `sitemap.md`, and the `.md` mirrors come from.

There are exactly two answers, and the framework does not choose between them:

| Mode | Condition | Mechanism | Files |
|------|-----------|-----------|-------|
| **Server** | A server runs at request time | One mounted handler | 1 |
| **Static** | Prerendered / CDN-only output | One prebuild script | 1 |

A SvelteKit app with `adapter-vercel` is server mode. The same app with `adapter-static` is static mode. Check the adapter and the output target, not the logo.

---

## Server Mode

One file per app. The handler owns every SEO path and content-negotiates Markdown on canonical URLs.

### Next.js

```typescript
// proxy.ts (project root)
import { createNextjsSeoProxy } from "@commercengine/seo/nextjs/server";
import { seo } from "@/lib/seo";

export default createNextjsSeoProxy(seo);
export const config = { matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"] };
```

Do **not** try to build these as route files. Two traps:

- `app/sitemap/[id].xml/` is not a dynamic segment — Next hands the route `params: {}` and it does not typecheck.
- Matchers are exact: `/search` does not match `/search.md`.

### Astro

```typescript
// src/middleware.ts
import { createAstroSeoMiddleware } from "@commercengine/seo/astro/server";
import { seo } from "./lib/seo";

export const onRequest = createAstroSeoMiddleware(seo);
```

Requires `output: "server"` (or `hybrid` with the SEO paths not prerendered) in `astro.config.mjs`.

### SvelteKit

```typescript
// src/hooks.server.ts
import { createSvelteKitSeoHandle } from "@commercengine/seo/sveltekit/server";
import { seo } from "$lib/commerce-seo";

export const handle = createSvelteKitSeoHandle(seo);
```

Compose with `sequence()` if you already have a handle.

### TanStack Start

```typescript
// src/start.ts
import { createStart } from "@tanstack/react-start";
import { createTanStackStartSeoMiddleware } from "@commercengine/seo/tanstack-start/server";
import { seo } from "@/lib/seo";

export const startInstance = createStart(() => ({
  requestMiddleware: [createTanStackStartSeoMiddleware(seo)],
}));
```

TanStack route files cannot express these paths anyway: route params must be valid JS identifiers, so `$slug[.]md` is rejected outright.

### What the handler serves

```
/robots.txt
/sitemap.xml                     (+ /sitemap/{n}.xml shards past 50k URLs)
/llms.txt
/sitemap.md
/product/{slug}.md
/category/{slug}.md
/search.md?q=...
/product/{slug}   with Accept: text/markdown   → the mirror, Vary: Accept
```

`HEAD` returns headers with an empty body. Non-`GET`/`HEAD` falls through to your app. A malformed percent-escape returns 400; an unknown `.md` path falls through rather than 404ing, so it cannot shadow your routes.

---

## Static Mode

One script, run before the framework build.

```javascript
// scripts/generate-seo-assets.mjs
import { existsSync, readFileSync } from "node:fs";
import { createCommerceSeo } from "@commercengine/seo";
import { writeCommerceSeoAssets } from "@commercengine/seo/build";
import { createStorefront, Environment } from "@commercengine/storefront";

// A local checkout keeps credentials in .env.local; CI and Vercel inject them into the
// process environment, where no such file exists. Prefer the file when present, fall back
// to the environment otherwise.
const envFile = new URL("../.env.local", import.meta.url);
const fileEnv = existsSync(envFile)
  ? Object.fromEntries(
      readFileSync(envFile, "utf8")
        .split("\n")
        .filter((line) => line.includes("=") && !line.trimStart().startsWith("#"))
        .map((line) => {
          const i = line.indexOf("=");
          // Values may be quoted; a literal quote in the store id produces a 404 that
          // looks like a credentials problem rather than a parsing one.
          return [line.slice(0, i).trim(), line.slice(i + 1).trim().replace(/^["']|["']$/g, "")];
        }),
    )
  : {};
const env = { ...process.env, ...fileEnv };

// Without credentials the catalog calls fail and this emits an empty sitemap that looks
// valid — fail the build with the missing names instead.
if (!env.PUBLIC_STORE_ID || !env.PUBLIC_API_KEY) {
  throw new Error(
    "[seo] missing PUBLIC_STORE_ID / PUBLIC_API_KEY. Set them in .env.local for a local " +
      "build, or in the deployment's environment variables.",
  );
}

const seo = createCommerceSeo({
  storefront: createStorefront({
    storeId: env.PUBLIC_STORE_ID,
    apiKey: env.PUBLIC_API_KEY,
    environment: env.PUBLIC_CE_ENV === "production" ? Environment.Production : Environment.Staging,
  }),
  site: { name: "Acme", url: "https://acme.example", brandName: "Acme" },
  routes: { productBase: "/product", categoryBase: "/category" },
  // A local build is not a production deployment, so detection would mark everything noindex.
  indexable: true,
});

const assets = await writeCommerceSeoAssets(seo, { outDir: "public" }); // "static" for SvelteKit
console.log(`[seo] wrote ${assets.length} assets`);
```

```json
{ "scripts": { "build": "node scripts/generate-seo-assets.mjs && vite build" } }
```

`outDir` is the directory the framework publishes verbatim: `public/` for Vite, Astro, and Next; `static/` for SvelteKit. `writeCommerceSeoAssets` refuses to write outside it.

### Gitignore the output

The assets are build artifacts. Ignore them, and **untrack any that were committed before you added the rule** — `.gitignore` has no effect on already-tracked files, so they keep reappearing in unrelated diffs:

```gitignore
# Generated by scripts/generate-seo-assets.mjs before each build.
/public/robots.txt
/public/sitemap.xml
/public/sitemap.md
/public/llms.txt
/public/product/
/public/category/
```

```bash
git rm --cached public/robots.txt          # then commit
git ls-files -i -c --exclude-standard      # must print nothing
```

### Vercel routing for static output

Two config gaps that produce 404s the build cannot detect:

- **SvelteKit `adapter-static`** emits `privacy-policy.html`. Without `cleanUrls`, Vercel serves it only at the `.html` path and every extensionless URL 404s — including all the URLs your sitemap advertises.

  ```json
  { "cleanUrls": true }
  ```

- **A single-page app** needs a catch-all rewrite or every deep link 404s on reload. Static files are matched before rewrites, so the `.md` mirrors and `llms.txt` still serve correctly.

  ```json
  {
    "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
  }
  ```

Verify against the deployment, not the build:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://acme.example/product/shoe
curl -s -o /dev/null -w "%{http_code}\n" https://acme.example/product/shoe.md
```

If `/foo` returns 404 while `/foo.html` returns 200, the pages deployed fine and only the routing is missing.

---

## Supporting Both

Nothing stops one codebase doing both — an app that prerenders at build time and also runs a server can mount the handler *and* ship the prebuild. The handler wins for negotiated requests; the files win for direct hits, because static files are matched before routing.

If you only ever deploy one way, ship one mechanism. Two is a maintenance cost with no benefit.
