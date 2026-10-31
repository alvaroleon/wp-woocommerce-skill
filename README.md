# wp-woocommerce — an agent skill for modern WooCommerce

A skill for AI coding agents (Claude Code, Codex, Cursor, Windsurf, GitHub Copilot and any agent that reads Markdown instructions) that writes and reviews WooCommerce code the way WooCommerce works today.

**Requirements: WooCommerce 11.2.1 or later · WordPress 7.0+ · PHP 8.4+.** The skill is written exclusively for that stack; it does not cover older WooCommerce versions or PHP releases.

Language models learned WooCommerce mostly from the shortcode/posts era. Left alone they write `woocommerce_checkout_fields`, `get_post_meta()` on orders, `wp_schedule_event()`, cart fragments, nonces printed into HTML and payment gateways that never appear in the Checkout block. This skill replaces those habits with what is true in current WooCommerce, verified against the **WooCommerce 11.2.1** source.

## Premises

The skill is opinionated. It targets one architecture and says so up front:

1. **Blocks only.** Cart, Checkout, Mini-Cart and product templates are blocks on the Store API, in a block theme. No shortcode checkout, classic checkout hooks, PHP template overrides or cart fragments.
2. **HPOS only.** Orders live in the `wc_orders*` tables and are accessed only through CRUD. No posts-storage fallbacks.
3. **Action Scheduler for all background work.** Never WP-Cron.
4. **Full-page HTML cache first.** Public pages are cacheable and identical for every visitor. Session, cart, customer data and nonces reach the browser asynchronously from the Store API.
5. **Every async request from the browser is a Store API request.** Use existing Store API routes, or develop new ones under `wc/store/v1`. Never `admin-ajax`, `wc-ajax` or a custom REST namespace for front-end calls.

If your project still runs the classic checkout or posts storage, this skill will steer you away from it on purpose.

## What's inside

```
SKILL.md                          Entry point: premises, "model biases to unlearn" table, router, boilerplate, red flags
references/
  cache-and-store-api.md          Full-page cache model, session cookies, runtime nonce, Mini-Cart, custom Store API routes
  checkout-blocks.md              Additional Checkout Fields API, Store API checkout hooks, checkout JS extensibility
  payments.md                     Gateways for the Checkout block, Payments settings page, money in minor units, webhooks
  shipping.md                     Shipping methods, rates, local pickup in blocks
  orders-hpos.md                  HPOS, compatibility declarations, meta boxes, wc_get_orders, custom statuses, lifecycle hooks
  products-data.md                CRUD, data stores, custom product types, enums, COGS, brands, render-time cache traps
  emails.md                       WC_Email, block email editor, deferred emails
  action-scheduler.md             Patterns, unique jobs, retries, batching, runners, dedicated workers
  rest-and-ai.md                  wc/v3 controllers and fields, webhooks, WooCommerce Abilities and MCP
  testing.md                      PHPUnit bootstrap with HPOS, wp-env, PHPStan stubs, E2E against the block checkout
  theme-css-variables.md          theme.json preset slugs and --wc-* CSS variables a block theme must provide
  releases.md                     9.0 → 11.2.1 timeline, feature flags, deprecations, what's next
evals/                            Prompts and grading notes used to compare this skill against a baseline
```

`SKILL.md` is short and always read first. It routes the agent to the one or two references a task needs, so the context stays small.

## Install

The skill uses the [Agent Skills](https://agentskills.io) layout: a folder with a `SKILL.md` (YAML frontmatter with `name` and `description`) plus supporting files. Agents with native skill support load it automatically when a task matches the description. For the others, add a small rule that points to `SKILL.md`.

### Claude Code

```bash
# Personal (all projects)
git clone https://github.com/alvaroleon/wp-woocommerce-skill.git ~/.claude/skills/wp-woocommerce

# Or per project, committed with the repo
git clone https://github.com/alvaroleon/wp-woocommerce-skill.git .claude/skills/wp-woocommerce
```

Claude Code discovers it on the next session; it triggers on WooCommerce work, or you can ask for it by name.

### Codex CLI and other agents that read `~/.agents/skills`

```bash
git clone https://github.com/alvaroleon/wp-woocommerce-skill.git ~/.agents/skills/wp-woocommerce
```

Codex also reads `~/.codex/skills/`. For a single repository, you can instead clone into the project and reference it from `AGENTS.md` (see "Any other agent" below).

### Cursor

Clone the skill into your project, for example to `.agents/skills/wp-woocommerce`, then add `.cursor/rules/woocommerce.mdc`:

```markdown
---
description: WooCommerce work (plugins, block themes, gateways, shipping, checkout, orders, Store API, Action Scheduler, page caching)
globs:
alwaysApply: false
---

Before writing or reviewing WooCommerce code, read `.agents/skills/wp-woocommerce/SKILL.md` and follow it,
then read only the files under `.agents/skills/wp-woocommerce/references/` that its router table points to for the task.
```

### Windsurf

Clone as above, then add `.windsurf/rules/woocommerce.md`:

```markdown
---
trigger: model_decision
description: WooCommerce work (plugins, block themes, gateways, shipping, checkout, orders, Store API, Action Scheduler, page caching)
---

Before writing or reviewing WooCommerce code, read `.agents/skills/wp-woocommerce/SKILL.md` and follow it,
then read only the references it routes you to.
```

### GitHub Copilot

Clone as above, then add `.github/instructions/woocommerce.instructions.md`:

```markdown
---
applyTo: "**/*.php,**/*.js,**/*.ts,**/*.tsx,**/theme.json,**/block.json"
---

For WooCommerce code, read `.agents/skills/wp-woocommerce/SKILL.md` first and follow it,
then the references it routes you to.
```

### Any other agent (`AGENTS.md`)

Most agents read an `AGENTS.md` at the repository root. Add:

```markdown
## WooCommerce

For any WooCommerce task, read `.agents/skills/wp-woocommerce/SKILL.md` before writing code and follow its premises.
Load only the references its router table lists for the task.
```

### Updating

```bash
git -C <path-to-the-skill> pull
```

## How it was verified

- Every API, hook, class and flag was checked against the WooCommerce 11.2.1 source (released 2026-10-09; requires WordPress 7.0 and PHP 7.4) and the official developer documentation. Claims that couldn't be verified are labelled as untested in the references.
- All PHP snippets pass `php -l` and use nothing deprecated in PHP 8.4. WooCommerce 11.2.1 itself still accepts PHP 7.4 (8.1 from 11.6); PHP 8.4 is this skill's own floor.
- The `evals/` folder holds seven realistic tasks. Each ran with the previous version of the skill and with this one, using the same model. With this version, answers stopped teaching the outdated patterns: they used the Mini-Cart block and the shared `storeApiNonce` instead of a hand-made cart badge, the `date` checkout field type, block email editor integration, and Action Scheduler with retries.

The code examples were reviewed against the source but **not executed on a live site**. Test before shipping, and open an issue if something is off.

## Contributing

Issues and pull requests are welcome, especially:

- corrections after a new WooCommerce release (`references/releases.md` lists what to re-check),
- evals that catch a model bias the skill doesn't cover yet.

Keep the premises intact: contributions that add classic-checkout, posts-storage or WP-Cron patterns won't be merged.

## License

[MIT](LICENSE)
