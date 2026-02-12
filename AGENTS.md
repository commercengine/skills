# AGENTS.md

Commerce Engine Skills is a collection of task-based guides and templates that help AI agents build e-commerce storefronts using the Commerce Engine SDK. Skills include decision trees for routing, code templates, and common pitfall documentation.

## Directory Structure

```
skills/
├── .claude-plugin/
│   └── marketplace.json      # Plugin registry metadata
├── commands/
│   └── ce.md                 # /ce slash command workflow
├── skills/
│   ├── ce/                   # Router skill
│   ├── setup/                # SDK setup & framework detection
│   ├── auth/                 # Authentication & user management
│   ├── catalog/              # Products, variants & categories
│   ├── cart-checkout/        # Cart, checkout & payments
│   ├── orders/               # Order management & returns
│   ├── webhooks/             # Webhook events & syncing
│   ├── subscriptions/        # Subscription management
│   └── nextjs-patterns/      # Advanced Next.js patterns
└── README.md                 # User-facing documentation
```

## Skill Routing

The master `skills/ce/SKILL.md` contains decision trees that route to the appropriate sub-skill:

```
User Request
    │
    ├─ "Set up SDK" / "Add Commerce Engine"  → setup/
    ├─ "Login" / "Auth" / "OTP"              → auth/
    ├─ "Products" / "Categories" / "Search"  → catalog/
    ├─ "Cart" / "Checkout" / "Payments"      → cart-checkout/
    ├─ "Orders" / "Returns" / "Shipments"    → orders/
    ├─ "Webhooks" / "Events" / "Sync"        → webhooks/
    ├─ "Subscriptions" / "Recurring"         → subscriptions/
    └─ "Next.js" / "SSR" / "Server Actions"  → nextjs-patterns/
```

### Skill Structure

Each skill follows this pattern:

```
skills/{skill-name}/
├── SKILL.md              # Main documentation (required)
├── .claude-plugin/
│   └── plugin.json       # Skill metadata (required)
├── references/           # Deep-dive docs (optional)
└── templates/            # Code templates (optional)
```

## Creating New Skills

### 1. SKILL.md Format

Every skill requires a `SKILL.md` with YAML frontmatter:

```yaml
---
name: ce-skill-slug
description: One-line description of what this skill does
license: MIT
metadata:
  author: commercengine
  version: "1.0.0"
---
```

### 2. Required Sections

Include these sections in every SKILL.md:

1. **Title & Overview** - What the skill does and when to use it
2. **Quick Start** - Table or list for rapid orientation
3. **Decision Tree** (if applicable) - ASCII flow chart for sub-routing
4. **Key Patterns** - Code examples for common use cases
5. **Common Pitfalls** - Mistakes agents make and how to avoid them
6. **See Also** - Cross-references to related skills

### 3. Template Guidelines

Templates should be:
- **Self-contained** - Copyable without modification for basic use
- **Framework-specific** - Organize by framework (nextjs/, react/, etc.)
- **Named clearly** - Indicate use case (e.g., `otp-login.tsx`)
- **Commented** - Explain non-obvious patterns inline

### 4. Impact Levels

Use these levels to prioritize guidance:

| Level | Meaning | Example |
|-------|---------|---------|
| CRITICAL | Breaking bugs, security holes | "Always call `sdk.auth.getAnonymousToken()` first" |
| HIGH | Common mistakes | "Use `CookieTokenStorage` for SSR, not `BrowserTokenStorage`" |
| MEDIUM | Optimizations | "Enable `NEXT_BUILD_CACHE_TOKENS` for faster builds" |
| LOW | Nice-to-have | "Cache categories client-side" |

## Key Patterns

### SDK Response Pattern

```typescript
// Every SDK call returns { data, error }
const { data, error } = await sdk.catalog.listProducts();

if (error) {
  console.error(error.message);
} else {
  console.log(data.products);
}
```

### Token Storage Selection

```typescript
// SPA (React, Vue, Svelte, Solid)
import { BrowserTokenStorage } from "@commercengine/storefront-sdk";
tokenStorage: new BrowserTokenStorage("myapp_")

// SSR (Next.js)
import { CookieTokenStorage } from "@commercengine/storefront-sdk";
tokenStorage: new CookieTokenStorage({ prefix: "myapp_" })

// Server-side (Node.js, Express)
import { MemoryTokenStorage } from "@commercengine/storefront-sdk";
tokenStorage: new MemoryTokenStorage()
```

### Product Hierarchy

```
Product (has_variant: false) → Simple product, one SKU
Product (has_variant: true)
  ├─ Variant A (Color: Red, Size: M) → SKU: "RED-M-001"
  └─ Variant B (Color: Blue, Size: L) → SKU: "BLU-L-001"
```

### Framework Detection

Check for these files to detect the framework:
- `next.config.js` or `next.config.mjs` → Next.js
- `vite.config.ts` with `@vitejs/plugin-react` → React SPA
- `vite.config.ts` with `@vitejs/plugin-vue` → Vue SPA
- `svelte.config.js` → Svelte/SvelteKit
- `solid-js` in package.json → Solid
- `express` in package.json → Express/Node.js

### Common Pitfalls (All Skills)

1. **Missing anonymous auth** - Must call `sdk.auth.getAnonymousToken()` before any API call
2. **Wrong token storage** - Use `CookieTokenStorage` for SSR, `BrowserTokenStorage` for SPA
3. **Product vs Variant vs SKU confusion** - Always check `has_variant` before accessing variant data
4. **Env var naming** - `CE_STORE_ID` and `CE_API_KEY` (or `NEXT_PUBLIC_*` for Next.js) must be set
5. **Cart expiration** - Carts have `expires_at`, handle gracefully

## Quality Checklist

Before submitting a new skill:

- [ ] SKILL.md has correct frontmatter (name, description, version)
- [ ] plugin.json has valid JSON with name, keywords, category
- [ ] Decision tree routes to correct sub-tasks (if applicable)
- [ ] Code examples use `{ data, error }` pattern
- [ ] Common pitfalls section includes agent-specific mistakes
- [ ] Cross-references to related skills in "See Also"
- [ ] All code examples use current Commerce Engine SDK patterns
- [ ] No hardcoded API keys in templates

## Resources

- [Commerce Engine Documentation](https://www.commercengine.io/docs)
- [LLM-Optimized API Reference](https://llm-docs.commercengine.io/)
- [SDK Reference](https://www.commercengine.io/docs/sdk/installation)
- [GitHub Issues](https://github.com/commercengine/skills/issues) - Request new skills

When adding a new skill, update `.claude-plugin/marketplace.json` if it adds new keywords or categories.
