# Evals for `wp-woocommerce`

Seven realistic store-extension prompts (Spanish, as the user types them) for WooCommerce 11.2.1 with HPOS, Cart/Checkout blocks and a full-page HTML cache. Evals 5–7 are built to tempt the model's outdated biases (server-rendered cart, WP-Cron, custom AJAX/nonce handling).

| ID | Name | Bias it probes | References it should load |
|---|---|---|---|
| 1 | `clp-gateway-block-checkout` | gateway without block integration, `*100` on CLP, webhook without HMAC | `payments.md`, `action-scheduler.md` |
| 2 | `checkout-block-extra-fields` | classic checkout field hooks | `checkout-blocks.md` |
| 3 | `hpos-order-metabox-query` | posts-storage fallbacks, `get_post_meta` | `orders-hpos.md` |
| 4 | `custom-status-email` | status not registered for HPOS, missing `woocommerce_email_actions`, no block email editor | `orders-hpos.md`, `emails.md` |
| 5 | `header-cart-count-free-shipping` | `WC()->cart` in templates, custom cart badge, own nonce key | `cache-and-store-api.md` |
| 6 | `erp-sync-background` | `wp_schedule_event`, inline ERP calls, `update_post_meta` | `action-scheduler.md`, `products-data.md` |
| 7 | `custom-block-store-api-js` | `admin-ajax`/`wc-ajax`, nonce in HTML, Mini-Cart not refreshed | `cache-and-store-api.md` |

## How to run

For each eval, run one subagent (same model for both arms, e.g. Sonnet) per arm and save its answer as markdown:

- **baseline**: the agent reads only the previous version of `SKILL.md` (keep a copy outside `~/.claude/skills`).
- **new**: the agent reads `~/.claude/skills/wp-woocommerce/SKILL.md` and the references it routes to.

Both arms: no web search, no WooCommerce source on disk, no access to this `evals/` folder.

## Grading

`expected_output` in `evals.json` lists what a correct answer contains. Grep the outputs for the red flags in `SKILL.md` (`WC()->cart` in render code, `wp_schedule_event`, `admin-ajax`, `wc-ajax`, `get_post_meta( $order`, `* 100`, `empty_cart`, custom `localStorage` nonce keys) and for the required APIs, then read every hit manually: quoted anti-patterns in "don't do this" notes are not violations.
