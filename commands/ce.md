---
description: Load Commerce Engine skill and get contextual guidance for any e-commerce task
---

Load the Commerce Engine skill and help with any storefront development task.

## Workflow

### Step 1: Check for --update-skill flag

If $ARGUMENTS contains `--update-skill`:

1. Run the update command:
   ```bash
   npx skills add commercengine/skills -y
   ```

2. Output success message and stop (do not continue to other steps).

### Step 2: Load ce skill

```
skill({ name: 'ce' })
```

### Step 3: Identify task type from user request

Analyze $ARGUMENTS to determine:
- **Task type**: setup, auth, catalog, cart/checkout, orders, webhooks, Next.js patterns
- **Platform**: Next.js, React, Vue, Svelte, Solid, Node.js, Vanilla JS

Use decision trees in SKILL.md to select correct skill.

### Step 4: Read relevant skill files

Based on task type, load the appropriate skill:

| Task | Skill | Files to Read |
|------|-------|---------------|
| SDK setup | `setup/` | SKILL.md |
| Auth / login | `auth/` | SKILL.md |
| Products / catalog | `catalog/` | SKILL.md |
| Cart / checkout | `cart-checkout/` | SKILL.md + relevant reference |
| Orders / returns | `orders/` | SKILL.md |
| Webhooks | `webhooks/` | SKILL.md |
| Next.js patterns | `nextjs-patterns/` | SKILL.md + relevant reference |

### Step 5: Look up exact method signatures when needed

**IMPORTANT**: Before generating any SDK code, verify the exact method signature from the LLM-optimized docs.

**Option A — Append `.md` extension** (works with any tool including WebFetch):

```
# Look up a specific SDK method signature
https://llm-docs.commercengine.io/storefront/operations/{operation-slug}.md

# Browse all SDK methods
https://llm-docs.commercengine.io/sdk/storefront.md

# Look up a schema (e.g., Cart, Product, Order)
https://llm-docs.commercengine.io/storefront/schemas/{SchemaName}.md

# Browse all webhooks
https://llm-docs.commercengine.io/webhooks/index.md

# Look up a specific webhook
https://llm-docs.commercengine.io/storefront/webhooks/{event-name}.md
```

**Option B — Use `curl` with custom header** (via Bash tool):

```bash
curl -s -H "Accept: text/markdown" https://llm-docs.commercengine.io/storefront/operations/{operation-slug}
curl -s -H "Accept: text/markdown" https://llm-docs.commercengine.io/sdk/storefront
curl -s -H "Accept: text/markdown" https://llm-docs.commercengine.io/storefront/schemas/{SchemaName}
curl -s -H "Accept: text/markdown" https://llm-docs.commercengine.io/webhooks/
```

Either approach returns markdown. Without `.md` extension or the `Accept: text/markdown` header, you'll get HTML instead of parseable markdown.

### Step 6: Execute task

Apply Commerce Engine patterns from skill to complete user's request.

### Step 7: Fallback for Unmatched Requests

If no skill matches the user's request:

1. Search Commerce Engine docs for relevant content:
   ```
   https://llm-docs.commercengine.io/sdk/storefront.md
   ```
2. If still unmatched, respond:

> I don't have a specific skill for this yet. Here are your options:
>
> 1. **Commerce Engine Docs**: [Search for "{topic}"](https://www.commercengine.io/docs)
> 2. **LLM-Optimized Reference**: [Browse API/SDK docs](https://llm-docs.commercengine.io/)
> 3. **Request a Skill**: [Create an issue](https://github.com/commercengine/skills/issues/new?template=skill-request.md&title=[SKILL]+{topic})
>
> Want me to help with the docs instead?

### Step 8: Summarize

```
=== Commerce Engine Task Complete ===

Skill: <skill used>
Framework: <detected framework>

<brief summary of what was done>
```

<user-request>
$ARGUMENTS
</user-request>
