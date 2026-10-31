# WooCommerce releases 9.0 → 11.2.1 (developer view)

Use this to answer "since which version?", to set `WC requires at least`, and to know which feature flags exist in 11.2.1. Only developer-relevant changes, verified against the 11.2.1 source or the official changelog. Everything under "Coming next" is **post-11.2.1** and not in the installed code.

## Timeline

| Version (date) | Developer-relevant change |
|---|---|
| 9.0 (2024-06-18) | Legacy REST API (`/wc-api/v1..v3`) replaced by a stub; separate plugin needed. `?wc-api=foo` → `woocommerce_api_foo` callbacks still fire. |
| 9.0 | Launch Your Store / coming-soon templates; `wc_product_pre_has_unique_sku` filter; experimental `hpos_fts_indexes`. |
| 9.1 (2024-07-10) | `woocommerce_coming_soon_exclude` filter; `woocommerce_order_note_deleted`; WP-CLI to toggle HPOS compatibility mode. |
| 9.2 (2024-08-20) | `doing_it_wrong` when HPOS-only query args hit a posts-table order query; Store API reserves stock only when payment is attempted; `woocommerce_plugins_are_incompatible_with_feature_by_default`; lazy REST namespace loading (`wc_rest_should_load_namespace`). |
| 9.3 (2024-09-10) | Store API: no nonce needed with a valid `Cart-Token`; deprecated `X-WC-Store-API-Nonce` header removed (use `Nonce`). `woocommerce_coming_soon` option for all sites; `woocommerce_should_clear_cart_after_payment`. |
| 9.4 (2024-11-11) | Product Brands in core (`product_brand` taxonomy; on for everyone in 9.6). `wc_rest_check_user_permissions()` multisite-aware. |
| 9.5 (2024-12-16) | All Products / Product List blocks hidden from the inserter; Block Hooks insert the Account Creation block. |
| 9.6 (2025-01-20) | `rate_limit_checkout` feature; coming-soon on by default for new installs; `FeatureGating` deprecated. |
| 9.8 (2025-04-07) | `woocommerce_register_feature_definitions` action deprecated; Store API loads the cart during `rest_api_init` for Store API requests. |
| 9.9 (2025-06-02) | Min WP 6.7. Product Filters blocks stable; block email templates; additional checkout fields get conditional `required`/`hidden` rules; `woocommerce_checkout_validate_order_before_payment`; `__internalSetExtensionData` → `setExtensionData`. |
| 10.0 (2025-07-07) | Old DI container removed (only `wc_get_container()`); empty session and its cookie removed automatically. |
| 10.1 (2025-08-12) | Core recurring jobs (sessions, unpaid orders, sales, logs) moved from WP-Cron to Action Scheduler; experimental Interactivity API Mini-Cart. |
| 10.2 (2025-09-17) | Product Filters taxonomy/hierarchical support; back-in-stock email templates; `wc/v4` controllers appear (behind a disabled flag). |
| 10.3 (2025-10-22) | Cost of Goods Sold stable; `add_feature_definition()` without `default_plugin_compatibility` → `doing_it_wrong`; experimental WooCommerce MCP (`mcp_integration`) with bundled MCP Adapter + Abilities API. |
| 10.4 (2025-12-10) | Min WP 6.8, PHP 7.4. `wc_enqueue_js()` deprecated → `wp_add_inline_script()`. Interactivity API Mini-Cart on by default. HPOS datastore caching stable. |
| 10.5 (2026-02-04) | PayPal Standard on Orders v2; `WC_Gateway_Paypal_*` helpers deprecated → `Automattic\WooCommerce\Gateways\PayPal\*`. `AccessiblePrivateMethods` trait removed. Experimental `rest_api_caching`. |
| 10.6 (2026-03-10) | Cart Store API responses send `Cache-Control: no-store`; WC Accordion → core Accordion. |
| 10.7 (2026-04-14) | Interactivity API cart store no longer trusts the server-rendered nonce: GETs `/wc/store/v1/cart` first and uses the `Nonce` header (fixes stale nonces on cached pages). HPOS sync-on-read off by default. |
| 10.8 (2026-05-26) | Min WP 6.9. `checkout-draft` orders excluded from REST `status=any`; `deferred_transactional_emails`; review-request email; webhook topic `product.published`. |
| 10.9 (2026-06-23) | Draft order created at place-order time (`woocommerce_store_api_checkout_update_draft`, `…_order_created`). Seven `woocommerce/*` domain abilities + public `AbilityDefinition` / `woocommerce_ability_definition_classes`. Webhook topic validation rejects hook-less default pairs. Product block editor deprecated. |
| 11.0 (2026-08-04) | Product block editor removed (feature, package, routes, extension APIs). Action Scheduler 4.0. Additional checkout fields warn when registered before `woocommerce_init`. `OrderUtil::custom_orders_table_data_sync_is_enabled()`. Order item removal deferred to `save()`. |
| 11.1 (2026-09-03) | Min WP 7.0. 30 stable wc-admin flags retired (`Features::is_enabled()` etc. are deprecated shims). Blocks not registered on REST/Store API/AJAX requests (`woocommerce_should_register_blocks`). Checkout returns 409 when the confirmed total ≠ server total. WC API keys rejected on routes outside `wc/`/`wc-`. `woocommerce_coupon_updated_props`. |
| 11.2 (2026-10-07) | `woocommerce_store_api_cart_item_quantity_validation`; Store API `item_data.raw_key`; `date` checkout field type with `min`/`max`. GraphQL Dual API, experimental Agentic Checkout API and abandoned-cart recovery removed. Action Scheduler 4.1. Settings registration moved to `admin_init` priority 999. Day-precision `wc_get_orders()`/`wc_get_products()` date queries anchored at store-local midnight. |
| 11.2.1 (2026-10-09) | Bug-fix release (classic checkout empty values, Checkout block postcodes). |

## Minimum WordPress / PHP

| From | WordPress | PHP |
|---|---|---|
| 9.2 | Required/tested bumped for WP 6.6 | — |
| 9.9 / 10.0 | 6.7 | — |
| 10.4 | 6.8 | 7.4 |
| 10.8 | 6.9 | 7.4 |
| 11.1 → 11.2.1 | 7.0 (readme: tested up to 7.1) | 7.4 |
| 11.6 (planned Feb 2027) | — | **8.1** (8.3 recommended); 11.5 is the last release for PHP 7.4/8.0 |

## Feature flags in 11.2.1

Stored in option `woocommerce_feature_{id}_enabled` unless the definition sets `option_key`. Query with `FeaturesUtil::feature_is_enabled( $id )`; deprecated features return their forced value and log a deprecation. **Compat** = what `FeaturesUtil::declare_compatibility( $id, __FILE__, true )` (inside `before_woocommerce_init`) does for that feature: **must** = undeclared plugins count as incompatible; **can** = compatibility checks run, default compatible, declaring is meaningful; **skip** = `skip_compatibility_checks`, declaring changes nothing. Declaring an unknown id (e.g. `product_block_editor`) returns `false` silently.

| id | Stage | Default | Compat | What |
|---|---|---|---|---|
| `custom_order_tables` | stable | off in definition; new shops auto-enabled by the installer | **must** | HPOS (option `woocommerce_custom_orders_table_enabled`) |
| `cart_checkout_blocks` | stable, UI hidden | — | can | Lists (in)compatible extensions in the Cart/Checkout block editor |
| `cost_of_goods_sold` | stable | off | can | COGS fields |
| `order_withdrawal` | stable (11.1) | off | can | Customer order withdrawal requests |
| `deferred_transactional_emails` | stable | off | can | Send emails from Action Scheduler |
| `fulfillments` | stable, UI hidden | off | can | Order fulfillments |
| `mcp_integration` | experimental | off | can | WooCommerce MCP (removed in 11.3) |
| `destroy-empty-sessions` | experimental | off | can | Drop the guest session cookie when empty |
| `cart_save_for_later` | experimental | off | can | Save-for-later list (option `woocommerce_cart_save_for_later_enabled`) |
| `product_wishlist` | experimental | off | can | Wishlists (option `woocommerce_product_wishlist_enabled`) |
| `product_instance_caching` | experimental | off (on for new installs since 11.0) | can | Per-request product object cache |
| `customer_stock_notifications` | experimental | off | can | Back-in-stock sign-ups |
| `block_editor_unified_assets` | experimental | off | can | Shared block editor bundles |
| `analytics` | stable | on | skip | WooCommerce Analytics |
| `rate_limit_checkout` | stable | off | skip | Rate-limit place order / Store API checkout |
| `order_attribution` | stable | on | skip | Order source attribution |
| `site_visibility_badge` | stable, UI hidden | on | skip | Coming-soon admin-bar badge |
| `hpos_fts_indexes` | experimental | off | skip | HPOS full-text indexes |
| `hpos_datastore_caching` | stable (10.4) | off | skip | HPOS order object cache |
| `remote_logging` | stable | on (needs tracking opt-in) | skip | Send logs to WooCommerce |
| `customer_review_request` | beta (not experimental) | off | skip | Review-request email + review page |
| `email_improvements` | stable | off in definition (on for new installs since 9.8) | skip | Modern email design |
| `blueprint` | beta label, not experimental | on | skip | Settings export/import |
| `block_email_editor` | experimental (alpha) | off | skip | Block-based email editor |
| `product_gallery_videos` | experimental | off | skip | Videos in the product gallery |
| `wc-visual-attribute` | experimental | off | skip | Color swatches (block themes) |
| `point_of_sale_staff` | experimental, UI hidden | off | skip | POS staff |
| `rest_api_caching` | experimental | off | skip | REST response cache + ETag/`X-WC-Cache` headers |
| `marketplace` | deprecated 10.5 | forced on | skip | In-app marketplace |
| `point_of_sale` | deprecated 11.0 | forced on | skip | Point of Sale |
| `push_notifications` | deprecated 10.9.2 | forced on | skip | Mobile push |
| `variation_gallery` | deprecated 11.1 | forced on | skip | Multiple images per variation |

Not feature flags: `product_block_editor` (removed 11.0); `rest-api-v4` is a wc-admin build flag, `false` in 11.2.1, so `wc/v4` is not registered.

## Deprecations and removals

| Thing | Status | Use instead |
|---|---|---|
| Legacy REST API `/wc-api/v1..v3`, webhooks on `legacy_v3` | Removed 9.0 (stub) | `wc/v3`; webhook API version `wp_api_v3` |
| `X-WC-Store-API-Nonce` header | Removed 9.3 | `Nonce` header (or `Cart-Token`) |
| `FeatureGating` class | Deprecated 9.6 | — |
| `woocommerce_register_feature_definitions` action | Deprecated 9.8 | — (core-only definitions) |
| `wc_current_theme_is_fse_theme()` | Deprecated 9.9 | `wp_is_block_theme()` |
| `woocommerce_product_availability` filter | Removed 9.9 | — |
| `__internalSetExtensionData` (checkout JS) | Renamed 9.9 | `setExtensionData` |
| Legacy DI container | Removed 10.0 | `wc_get_container()` (WooCommerce's own services only) |
| `add_feature_definition()` without `default_plugin_compatibility` | `doing_it_wrong` since 10.3 | Pass it |
| `wc_enqueue_js()` | Deprecated 10.4 (notice since 10.5) | `wp_add_inline_script()` |
| `WC_Gateway_Paypal_{Helper,Notices,Buttons,Constants,Webhook_Handler,Transact_Account_Manager}` | Deprecated 10.5 | `Automattic\WooCommerce\Gateways\PayPal\*` |
| `AccessiblePrivateMethods` trait | Removed 10.5 | — |
| `CartCheckoutUtils::is_overriden_by_custom_template_content()` | Renamed 10.9 (alias kept) | `is_overridden_by_custom_template_content()` |
| Product block editor, `@woocommerce/product-editor`, block templates PHP API, `declare_compatibility( 'product_block_editor' )` | Removed 11.0 (`BlockRegistry` no-op shim) | Classic product data tabs (`products-data.md`) |
| `Automattic\WooCommerce\Blocks\QueryFilters` | Deprecated 11.0 | — |
| `Features::is_enabled()/exists()/enable()/disable()` for retired wc-admin flags; `woocommerce_admin_features` to disable features | Deprecated 11.1 | Retired flags are always on |
| `woocommerce_coupon_object_updated_props` | Deprecated 11.1 | `woocommerce_coupon_updated_props` |
| `woocommerce_after_single_product_ordering`, `woocommerce_after_product_ordering` | Deprecated 11.2 (fall back to the slow algorithm) | `woocommerce_product_ordering_process_reindexed_products` / `…_moved_products` |
| `WC_Brands` recount methods | Deprecated 11.2 | — |
| GraphQL Dual API, Agentic Checkout API (experimental), abandoned-cart recovery (experimental), Activity panel | Removed 11.2 | Dual API → separate plugin |
| `woocommerce_update_cart_validation`, `woocommerce_add_to_cart_validation` | Classic/shortcode only (not deprecated) | `woocommerce_store_api_cart_item_quantity_validation` (11.2), `woocommerce_store_api_validate_add_to_cart` |

## Coming next (post-11.2.1 — not in the installed code)

- **11.3** (beta; release planned the week of 2026-11-09):
  - **Cart item children:** filter `woocommerce_store_api_cart_item_parent_item_key( $parent_item_key, $cart_item )` and read-only Store API cart item field `parent_item_key` (string or `null`). Return the parent's cart item key (from `WC()->cart->add_to_cart()`, not a product id) for items added on behalf of another item (bundles, composites, force-sells). The Product Button "N in cart" count then skips children. WooCommerce never marks children itself; an unknown key or the item's own key resolves to `null`.
  - **MCP:** `/wp-json/woocommerce/mcp` returns 410; `X-MCP-API-Key`, the `mcp_integration` flag, the REST-derived abilities, `woocommerce_mcp_include_ability`, `woocommerce_mcp_allow_insecure_transport`, `woocommerce_check_rest_ability_permissions_for_method` and `expose_in_deprecated_woocommerce_mcp` are removed; MCP Adapter and Abilities API packages are no longer bundled. Use the MCP Adapter plugin + Application Passwords (`rest-and-ai.md`).
  - Experimental, off by default: extensible abilities (`Automattic\WooCommerce\Abilities\ExtensibleAbilities`, fields under `extensions.<namespace>`); API still changing.
  - New core ability `woocommerce/order-notes-query`.
  - Dismissible admin notice on PHP 7.4/8.0.
- **11.5:** last release that runs on PHP 7.4 / 8.0.
- **11.6** (planned February 2027): **PHP 8.1 minimum**. New extensions can target PHP 8.1 syntax unless they must support WooCommerce ≤ 11.5.
