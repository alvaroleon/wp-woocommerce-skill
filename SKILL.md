---
name: wp-woocommerce
description: Use when writing, reviewing or planning WooCommerce code (WooCommerce 11.2.1+, PHP 8.4+) — plugins, block themes, payment gateways, shipping methods, checkout fields, cart behaviour, order data, custom order statuses, transactional emails, product types, Store API or REST extensions, background jobs, WooCommerce tests — and whenever a task involves the cart, checkout, orders, HPOS, the Store API, Mini-Cart, nonces, full-page caching of a shop, Action Scheduler or WP-Cron in a WooCommerce site.
---

# WooCommerce (blocks + HPOS) — Router

**Target: WooCommerce 11.2.1 or later, WordPress 7.0+, PHP 8.4+.** Verified against the WooCommerce 11.2.1 source (2026-10-09). WooCommerce itself still accepts PHP 7.4 (8.1 from 11.6); PHP 8.4 is this skill's floor, and its code examples avoid anything deprecated in 8.4. This skill exists to replace what a language model "remembers" about WooCommerce — mostly the 2018–2023 shortcode/posts era — with how WooCommerce works now.

## Premises (non-negotiable — this skill is built for them)

1. **Blocks only.** Cart, Checkout, Mini-Cart, Order Confirmation and product templates are blocks on the Store API, in a block theme. Nothing classic: no shortcode cart/checkout, no `woocommerce_checkout_*` hooks, no PHP template overrides, no cart fragments, no `admin-ajax`/`wc-ajax` cart calls. Classic names appear in this skill only as "don't use → use instead".
2. **HPOS only.** Orders live in the `wc_orders*` tables and are touched only through CRUD (`wc_get_order()`, `wc_get_orders()`, `$order->get_meta()/update_meta_data()/save()`). No posts-storage fallbacks, no `'shop_order'` screen ids, no `OrderUtil` if/else branches.
3. **Action Scheduler for all background and scheduled work.** `as_enqueue_async_action`, `as_schedule_single_action`, `as_schedule_recurring_action`, `as_schedule_cron_action`. Never `wp_schedule_event()`/`wp_schedule_single_event()`. WooCommerce moved its main recurring jobs (sales, unpaid orders, sessions, logs) to Action Scheduler in 10.1; the few WP-Cron users left in core are not a pattern to copy.
4. **Full-page HTML cache first.** Public pages (home, category, product) are served from a page/edge cache identical for every visitor. Session, cart, customer and nonces reach the browser asynchronously from the Store API; nothing per-user is rendered into HTML. Cart, checkout and my-account are never cached.
5. **Every async request from the browser is a Store API request.** Use an existing Store API route (`/wp-json/wc/store/v1/…`: cart, checkout, products) or develop a new one in the Store API — for anything the front end fetches asynchronously, cart-related or not (availability, a form submit, a lookup, a block's data). Never `wp_ajax_*`/`wp_ajax_nopriv_*`, `admin-ajax.php`, `wc-ajax`, or a separate REST namespace. Extra data travels in the `extensions` payload (`woocommerce_store_api_register_endpoint_data`); extra cart actions go through `cart/extensions` (`extensionCartUpdate`). A custom public endpoint is registered under the `wc/store/v1` namespace (`register_rest_route( 'wc/store/v1', '/my-plugin/…' )`): any route starting with `/wc/store/` is treated by core as a Store API request, so it gets the `Nonce`/`Cart-Token` CORS headers and the Cart-Token session handler (rate limiting too, when enabled). Core does not load the cart, check the `Nonce` or set `Cache-Control` on your routes: call `wc_load_cart()` when you need the shopper session or cart, verify the nonce yourself on writes (CSRF), and every route sets its own cache headers. Where `wp.apiFetch` with core's `wc-blocks-middleware` is loaded, it adds the `Nonce` header to any non-GET call to `wc/store/v1/`, your routes included. `wc/v3` (REST API) is for back-office integrations with API keys, not for the storefront.

The one surface with no block alternative: **product admin**. The block product editor was removed in 11.0 with no replacement, so extending the product form still means the product data tabs (`woocommerce_product_data_tabs`, `woocommerce_admin_process_product_object`). See `references/products-data.md`.

## Model biases to unlearn

Each row is a belief a model typically brings, and what is true in 11.2.1. If your draft matches the left column, stop and use the right one.

| Outdated belief | Reality in WooCommerce 11.x |
|---|---|
| Add checkout fields with `woocommerce_checkout_fields`, validate in `woocommerce_checkout_process`, save in `woocommerce_checkout_update_order_meta` | None of those fire in the Checkout block. Use `woocommerce_register_additional_checkout_field()` (on `woocommerce_init`; types `text`, `select`, `checkbox`, `date`), its `validate_callback`/`sanitize_callback`, and Store API hooks. Values show up in admin, emails and My Account automatically. |
| Validate the checkout with `woocommerce_store_api_register_update_callback` | That only serves `POST /cart/extensions` (`extensionCartUpdate`). Checkout validation: field `validate_callback`, `woocommerce_checkout_validate_order_before_payment( $order, WP_Error $errors )`, or throw `RouteException` in `woocommerce_store_api_checkout_update_order_from_request`. |
| A draft order exists as soon as the shopper reaches checkout | Since 10.8/10.9 the draft order is created at place-order time (`woocommerce_store_api_checkout_update_draft`, `…_order_created`). Since 11.1 checkout returns 409 `woocommerce_rest_checkout_total_mismatch` when the server total is higher than the `expected_total` the client sent. |
| Throw an exception in a checkout field `validate_callback` | Exceptions there are caught and checkout continues; return a `WP_Error`. |
| A `WC_Payment_Gateway` subclass is enough | It never appears in the Checkout block without an `AbstractPaymentMethodType` registered on `woocommerce_blocks_payment_method_type_registration` plus a JS `registerPaymentMethod()`. Use `onPaymentSetup` (`onPaymentProcessing` is deprecated); read server data with `getPaymentMethodData( name )`. |
| `'amount' => $order->get_total() * 100` | Float math, and 100× overcharge for zero-decimal currencies (CLP, JPY…). Convert with the provider's currency exponent; compare amount + currency before `payment_complete()`. |
| Call `WC()->cart->empty_cart()` in `process_payment()` | Don't. Core clears the cart on the order-received page; emptying it inside the Store API request breaks retries. On failure return `['result' => 'failure', 'message' => …]`. |
| Webhook on `woocommerce_api_{class}` that processes inline and ends with `wp_die('OK')` | REST route, HMAC on the raw body with `hash_equals`, dedupe by event id, enqueue a unique Action Scheduler job, return JSON 200 fast. Refunds via `wc_create_refund()`, not `update_status('refunded')`. |
| Schedule sync jobs with `wp_schedule_event` + `cron_schedules` | Action Scheduler. Schedule recurring actions on activation and re-assert them on `action_scheduler_ensure_recurring_actions` (with `$unique = true`); pass ids in args (8000-char JSON limit, positional args); failed single/async actions are never retried — reschedule with backoff yourself. |
| Run ERP/CRM calls inside `woocommerce_payment_complete` / status hooks | Enqueue `as_enqueue_async_action( 'x', [ $order_id ], 'my-plugin', true )` and do the work in the job. |
| Show the cart count with `WC()->cart->get_cart_contents_count()` in a template, refresh with cart fragments | Use the **Mini-Cart block** (Interactivity API, the only implementation since 11.1). Its store does `GET /wc/store/v1/cart` on load and keeps itself in sync; cached empty-cart HTML is correct for everyone. Never read `WC()->cart`/`WC()->session` while rendering. |
| Build your own header cart badge / add-to-cart with `admin-ajax` | Product Button / Add to Cart + Options / Mini-Cart blocks. For a custom button: `POST /wc/store/v1/cart/add-item` from JS, then dispatch `wc-blocks_added_to_cart` so the Mini-Cart refreshes. |
| Print a nonce with `wp_create_nonce()` / `wp_localize_script()` for the Store API | The nonce is a runtime value: read it from each response's `Nonce` header, persist it as `localStorage.storeApiNonce` in core's JSON format `{"nonce":"…","timestamp":…}` (a plain string breaks interop with core's middleware), send it as the `Nonce` header on writes. A GET to `/wc/store/v1/cart` gives you a fresh one. |
| Store a Cart-Token in `localStorage` to keep the cart | The session is the HttpOnly cookie `wp_woocommerce_session_*`, sent with `credentials: 'same-origin'`. A custom Cart-Token creates a parallel session that diverges from `/cart` and `/checkout`. The client cart state is gated by the `woocommerce_items_in_cart` and `woocommerce_cart_hash` cookies. |
| Async front-end requests go to `wp_ajax_*`/`admin-ajax.php`, `wc-ajax`, or a `my-plugin/v1` REST namespace | Store API: extend core responses with `woocommerce_store_api_register_endpoint_data`, mutate via `extensionCartUpdate`, and register custom shopper routes under `wc/store/v1` so they share the session, Cart-Token and CORS handling (load the cart with `wc_load_cart()` and verify the nonce yourself on writes). |
| `woocommerce_add_to_cart_validation` + `wc_add_notice()` | `woocommerce_store_api_validate_add_to_cart( $product, $request )` (throw `RouteException`); quantity changes: `woocommerce_store_api_cart_item_quantity_validation` (11.2, return `WP_Error`). |
| HPOS is optional; branch on `OrderUtil::custom_orders_table_usage_is_enabled()` | HPOS is the default for new shops and the only storage this skill targets. `wc_get_orders()` already uses the active store. Meta boxes on `wc_get_page_screen_id( 'shop-order' )`; the callback receives a `WC_Order`. |
| `get_post_meta()` / `WP_Query( post_type=shop_order )` / SQL on `wp_posts` for orders | CRUD only. `wc_get_orders()` supports `meta_query`, `field_query` (order columns) and `date_query`. |
| `wc_get_products( [ 'meta_query' => … ] )` filters by meta | Ignored. Add custom vars with `woocommerce_product_data_store_cpt_get_products_query`; brands via `tax_query` on `product_brand`. |
| Register a custom order status with `register_post_status()` only | Register it in `woocommerce_register_shop_order_post_statuses` **and** `wc_order_statuses`; slug ≤ 20 chars including `wc-`. Add the transition to `woocommerce_email_actions` or its `_notification` hook never fires. |
| Define `class X extends WC_Email` at plugin load | Include the class inside the `woocommerce_email_classes` callback; send with `send_notification()`; for the block email editor (alpha, off by default) opt in with `woocommerce_transactional_emails_for_block_editor`. With deferred emails on (off by default) they run in Action Scheduler, so email args must be ids/WC objects. |
| Product block editor / `@woocommerce/product-editor` / `declare_compatibility( 'product_block_editor' )` | Removed in 11.0. The declaration silently returns false. Declare `custom_order_tables` (mandatory) and `cart_checkout_blocks`. |
| Save product fields on `woocommerce_process_product_meta_{type}` | Fires after core saved → second save. Use `woocommerce_admin_process_product_object( WC_Product $product )`. |
| `/wc-api/v3` legacy REST API | Removed in 9.0. Use `wc/v3` (REST) and `wc/store/v1` (Store API). `wc/v4` is shipped but disabled. |
| `WC_Unit_Test_Case` / `WC_Helper_Order` are available | Not shipped in the plugin zip; bootstrap tests yourself (see `references/testing.md`). |
| `wc_enqueue_js()` | Deprecated 10.4 → `wp_add_inline_script()`. |
| `$product->get_price_html()` is always cache-safe | Only if every visitor resolves to the same tax location: default customer address = shop base, no geolocation. |

## Which reference to load

| Task | Read |
|---|---|
| Anything that touches the cart from the browser, nonces, sessions, Mini-Cart, page caching, Interactivity API cart store, custom JS against the Store API, `ExtendSchema` (`woocommerce_store_api_register_endpoint_data`), `extensionCartUpdate`, fees | `references/cache-and-store-api.md` |
| Checkout fields, checkout validation, Store API checkout hooks, checkout/cart JS extensibility (`IntegrationInterface`, `registerCheckoutBlock`, `registerCheckoutFilters`, slot fills, `wc/store/*` data stores) | `references/checkout-blocks.md` |
| Payment gateways: contract, Checkout block integration, Payments settings page, money in minor units, webhooks, refunds, tokens | `references/payments.md` |
| Shipping methods, rates, local pickup in blocks | `references/shipping.md` |
| Orders on HPOS: compatibility declarations, `OrderUtil`, meta boxes, orders list columns, `wc_get_orders`, custom statuses, order lifecycle hooks, fulfillments | `references/orders-hpos.md` |
| Products and CRUD: data stores, custom product types, enums, COGS, brands, product queries, render-time cache traps | `references/products-data.md` |
| Transactional emails, block email editor, deferred emails | `references/emails.md` |
| Background work: Action Scheduler patterns, retries, batching, runners, dedicated worker | `references/action-scheduler.md` |
| REST `wc/v3` controllers and fields, webhooks, WooCommerce Abilities and MCP | `references/rest-and-ai.md` |
| Tests, wp-env, PHPStan stubs, sniffs, E2E against the block checkout | `references/testing.md` |
| Building a block theme for a shop: `theme.json` preset slugs and `--wc-*` CSS variables that WooCommerce templates, patterns and block styles expect | `references/theme-css-variables.md` |
| "Since which version?", feature flags, deprecations and removals 9.0 → 11.2.1, what's coming in 11.3+ | `references/releases.md` |

Load only what the task needs. Most tasks: this file + one or two references.

## Plugin boilerplate (every WooCommerce extension)

```php
<?php
/**
 * Plugin Name:       My Plugin
 * Requires at least: 7.0
 * Requires PHP:      8.4
 * Requires Plugins:  woocommerce
 * WC requires at least: 11.2.1
 * WC tested up to:   11.2
 * Text Domain:       my-plugin
 */

defined( 'ABSPATH' ) || exit;

add_action( 'before_woocommerce_init', static function () {
	// Must run in before_woocommerce_init. custom_order_tables is the only feature that treats
	// undeclared plugins as incompatible; cart_checkout_blocks keeps the block editor from flagging you.
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'cart_checkout_blocks', __FILE__, true );
} );

add_action( 'woocommerce_init', static function () {
	// Register checkout fields, product types, gateways' dependencies here — not at file load.
} );
```

Use public APIs only: `wc_*` functions, CRUD objects, `Automattic\WooCommerce\Utilities\*`, `Automattic\WooCommerce\Enums\*`, Store API functions. Anything under `Automattic\WooCommerce\Internal\*` or tagged `@internal` has no backward-compatibility promise, and `wc_get_container()` cannot hold your own services.

## Red flags — stop and re-read the matching reference

- `WC()->cart`, `WC()->session`, `WC()->customer` or `wp_create_nonce()` in a template, `render.php` or `render_callback`.
- `wp_schedule_event`, `cron_schedules`, `WC()->queue()` (no `$unique`), or an HTTP call to a third party inside a checkout/status hook.
- `woocommerce_checkout_*`, `woocommerce_after_order_notes`, `woocommerce_form_field`, `[woocommerce_checkout]`, `wc-cart-fragments`, `admin-ajax.php?action=woocommerce_add_to_cart`.
- `get_post_meta( $order_id`, `'post_type' => 'shop_order'`, `'shop_order'` as a screen id, `$wpdb->posts` for orders.
- `* 100` on a money value, `empty_cart()` in a gateway, `wp_die()` in a webhook.
- A custom `localStorage` key for the nonce or a Cart-Token.
- `wp_ajax_*`, `wp_ajax_nopriv_*`, `admin-ajax.php`, `wc-ajax`, or `register_rest_route( 'my-plugin/v1', …)` for anything the browser fetches asynchronously.
- `declare_compatibility( 'product_block_editor'`, `@woocommerce/product-editor`, `wc_enqueue_js`.
