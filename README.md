<p align="center">
  <a href="https://www.commercengine.io?utm_source=github&utm_medium=ce_skills" target="_blank" rel="noopener noreferrer">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://images.tarkai.com/Commerce_Engine/Brand%20Assets/01%20-%20Logo/svg/CE_logo_darkbg.svg">
      <img src="https://images.tarkai.com/Commerce_Engine/Brand%20Assets/01%20-%20Logo/svg/CE_logo_whitebg.svg" height="48" alt="Commerce Engine">
    </picture>
  </a>
  <br />
</p>
<div align="center">
  <h1>
    Commerce Engine Skills
  </h1>
  <a href="https://www.commercengine.io/docs">
    <img alt="Documentation" src="https://img.shields.io/badge/documentation-commercengine-00bf6f.svg" />
  </a>
  <br />
  <br />
  <p>
    <strong>
      Skills to help AI coding agents build e-commerce storefronts with Commerce Engine.
    </strong>
  </p>
</div>

---

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Install

```bash
npx add-skill commercengine/skills
```

### Alternative Installation

```bash
# Using Vercel's skills CLI
npx skills add commercengine/skills

# Manual (Claude Code)
git clone https://github.com/commercengine/skills ~/.claude/skills/commercengine
```

## Skills

| Skill | Purpose | When to Use | Type |
|-------|---------|-------------|------|
| `/ce` | **CE router** - Routes to the right skill | Always start here | Router |
| `ce-setup` | SDK setup & framework detection | New projects, SDK install | Setup |
| `ce-auth` | Authentication & user management | Login, OTP, anonymous auth | Auth |
| `ce-catalog` | Products, variants & categories | Product listing, search, reviews | Catalog |
| `ce-cart-checkout` | Cart, checkout & payments | Cart CRUD, checkout flow, hosted checkout | Cart & Checkout |
| `ce-orders` | Order management & returns | Order history, tracking, returns | Orders |
| `ce-webhooks` | Webhook events & syncing | Real-time events, notifications | Webhooks |
| `ce-nextjs-patterns` | Advanced Next.js patterns | SSR, Server Actions, SSG | Framework Patterns |

## Quick Start

### 1. Get API Credentials

Get your Store ID and API Key from the [Commerce Engine Dashboard](https://app.commercengine.io).

### 2. Set Environment Variables

```bash
# For Next.js
NEXT_PUBLIC_STORE_ID=your-store-id
NEXT_PUBLIC_API_KEY=your-api-key

# For other frameworks
CE_STORE_ID=your-store-id
CE_API_KEY=your-api-key
```

### 3. Ask Your Agent

| You Say | Skill Used |
|---------|------------|
| "Set up Commerce Engine in my Next.js app" | `ce-setup` |
| "Add email OTP login" | `ce-auth` |
| "Build a product listing page" | `ce-catalog` |
| "Add cart and checkout" | `ce-cart-checkout` |
| "Show order history with tracking" | `ce-orders` |
| "Set up webhooks for order events" | `ce-webhooks` |
| "Use Server Actions for cart mutations" | `ce-nextjs-patterns` |

## Repository Structure

```
commerce-engine-skills/
├── .claude-plugin/
│   └── marketplace.json         # Plugin registry
├── skills/
│   ├── ce/                      # Router skill
│   │   └── SKILL.md
│   ├── setup/                   # SDK setup
│   │   └── SKILL.md
│   ├── auth/                    # Authentication
│   │   └── SKILL.md
│   ├── catalog/                 # Products & categories
│   │   └── SKILL.md
│   ├── cart-checkout/           # Cart & checkout
│   │   ├── SKILL.md
│   │   └── references/
│   ├── orders/                  # Order management
│   │   └── SKILL.md
│   ├── webhooks/                # Webhook events
│   │   └── SKILL.md
│   └── nextjs-patterns/         # Next.js patterns
│       ├── SKILL.md
│       └── references/
└── README.md
```

## Using /ce Command

For agents that support slash commands (Claude Code, OpenCode):

```
/ce set up SDK in my React app
/ce add email OTP login
/ce build checkout flow with Juspay payments
```

## Resources

- [Commerce Engine Docs](https://www.commercengine.io/docs)
- [LLM-Optimized API Reference](https://llm-docs.commercengine.io/)
- [SDK Installation](https://www.commercengine.io/docs/sdk/installation)
- [Dashboard](https://dashboard.commercengine.io)

## Request a Skill

Don't see what you need? [Request a skill](https://github.com/commercengine/skills/issues/new?template=skill-request.md).

## License

MIT
