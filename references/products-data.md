# Products and the CRUD data layer

How to read, write and query products, how to build a custom product type end to end, and what in the data layer is or isn't safe behind a full-page HTML cache. WooCommerce 11.2.1.

Products are `WC_Data` objects. Change them through getters, setters and `save()`, never with `update_post_meta()` or SQL. Product admin is the one surface with no block alternative: the block product editor was removed in 11.0, so extending the product form still means the classic product data tabs.

## `WC_Data` CRUD essentials

All CRUD objects (`WC_Product`, `WC_Product_Variation`, `WC_Order`, `WC_Coupon`, `WC_Customer`, order items) share `includes/abstracts/abstract-wc-data.php`:

- `save()` fires `woocommerce_before_{object_type}_object_save( $object, $data_store )`, then the store's `create()` or `update()`, then `woocommerce_after_{object_type}_object_save`. It returns the id. Object types include `product`, `product_variation`, `order` and `coupon`.
- Setters and meta changes are **buffered** until `save()`. `get_changes()` lists pending prop changes. `set_props( array )` returns `true` or a `WP_Error`.
- Meta: `get_meta( $key, $single = true, $context = 'view' )` (the `view` context applies filters), `meta_exists()`, `add_meta_data( $k, $v, $unique = false )`, `update_meta_data( $k, $v, $meta_id = 0 )`, `delete_meta_data( $k )`, `get_meta_data()`.
- **Internal meta keys** (those with a getter/setter, such as `_price`, `_stock`, `_sku`) must go through `get_*()`/`set_*()`. Using the generic meta methods on them triggers `_doing_it_wrong`.
- `delete( $force = false )`. `woocommerce_pre_delete_{object_type}` can short-circuit it.
- Clone modes (10.4): `WC_Data::CLONE_MODE_DUPLICATE` (default, clears meta ids) and `CLONE_MODE_CACHE`. They only matter if you cache WC objects yourself, which you shouldn't.
- Save once per flow. Every `save()` writes, clears caches and fires hooks that other plugins listen to.

## Data stores

- The `WC_Data_Store` registry has keys such as `product`, `product-variable`, `product-grouped`, `product-variation`, `order`, `order-refund`, `customer`, `coupon`, `webhook` and the order item stores. Override with `woocommerce_data_stores` (all) or `woocommerce_{object_type}_data_store` (one). A replacement must implement `WC_Object_Data_Store_Interface`.
- A product loads `WC_Data_Store::load( 'product-' . $type )`. An unregistered `product-{type}` key **falls back to `product`** (`WC_Product_Data_Store_CPT`). A custom type gets a store only if you register `product-{type}`. Never replace `product`, which swaps the store for every product.
- Overriding `order` through `woocommerce_data_stores` **has no effect under HPOS**: the HPOS controller injects `OrdersTableDataStore` at `woocommerce_order_data_store` priority 999. Don't replace the order store.

## Retrieving products

- `wc_get_product( $product = false )` returns `false` (with `_doing_it_wrong`) if called before `woocommerce_init` and the product taxonomies/post types are registered. Call it from `init` or later. The second parameter is deprecated.
- `wc_get_product_object( $type, $id = 0 )` instantiates the right class for a type. Use it to create products: `$p = wc_get_product_object( 'simple' ); $p->set_name( … ); $p->save();`.
- `wc_get_products( $args )` → `WC_Product_Query`. The `type` default is every key of `wc_get_product_types()` (variations excluded: add `'variation'` to the array to include them). `limit` defaults to `posts_per_page`. `return` is `'objects'` or `'ids'`. Supported: `status`, `include`, `exclude`, `parent`, `sku`, `category`, `tag`, `featured`, `visibility`, `stock_status`, `manage_stock`, price and date args, `orderby`/`order`, `page`, `paginate`.
- **`meta_query` passed to `wc_get_products()` is ignored.** `WC_Data_Store_WP::get_wp_query_args()` skips the key. Add your own query var and translate it with `woocommerce_product_data_store_cpt_get_products_query( $wp_query_args, $query_vars, $data_store )`.
- Unknown keys (e.g. `tax_query`) pass through to `WP_Query` and merge with WooCommerce's own `tax_query`. **There is no `brand` arg**: filter brands with `tax_query` on `product_brand`.
- 11.2: day-precision dates (`'date_created' => '2026-10-01'`, `'a...b'`) are anchored at store-local midnight, DST-aware, and `a...b` includes the full last day.

```php
// A custom query var: wc_get_products( [ 'my_plugin_origin' => 'itata' ] ).
add_filter( 'woocommerce_product_data_store_cpt_get_products_query', 'my_plugin_origin_query_var', 10, 2 );
function my_plugin_origin_query_var( $wp_query_args, $query_vars ) {
	if ( ! empty( $query_vars['my_plugin_origin'] ) ) {
		$wp_query_args['meta_query'][] = array(
			'key'   => '_my_plugin_origin',
			'value' => sanitize_text_field( $query_vars['my_plugin_origin'] ),
		);
	}
	return $wp_query_args;
}

// Brands: tax_query on product_brand.
$ids = wc_get_products(
	array(
		'status'    => 'publish',
		'limit'     => 12,
		'return'    => 'ids',
		'tax_query' => array(
			array(
				'taxonomy' => 'product_brand',
				'field'    => 'slug',
				'terms'    => array( 'my-brand' ),
			),
		),
	)
);
```

## Custom product type, end to end

Extend **`WC_Product_Simple`** when the type is bought like a simple product. A bare `WC_Product` subclass has `add_to_cart_text()` = "Read more", `add_to_cart_url()` = the permalink and no `ajax_add_to_cart` support. Without that support the Product Button block can't add it to the cart through the Store API.

| Step | Hook / API |
|---|---|
| Class | `class My_Plugin_Product_Gift_Box extends WC_Product_Simple`, `get_type()` returns the slug |
| Factory | filter `woocommerce_product_class( $classname, $type, $context, $product_id )`. Without it the default class name would be `WC_Product_Gift_Box`, and a missing class falls back to `WC_Product_Simple`. |
| Selector | filter `product_type_selector`. This list also feeds `wc_get_product_types()` and therefore the default `type` of `wc_get_products()`. |
| Admin tab + panel | `woocommerce_product_data_tabs`, `woocommerce_product_data_panels` (the non-block surface, see below) |
| Save | `woocommerce_admin_process_product_object( WC_Product $product )` |
| Front-end form | action `woocommerce_{type}_add_to_cart` |
| Add-to-cart handler | none needed: unknown types use the simple handler |

**Product admin is the one non-block surface.** The block product editor was deprecated in 10.9 and removed in 11.0 with no replacement (`@woocommerce/product-editor` and `declare_compatibility( 'product_block_editor' )` are gone). In 11.2.1 the only way to extend the product form is the classic product data meta box: tabs, panels, `woocommerce_wp_*` field helpers and `woocommerce_product_options_*` hooks.

**Save order** (`WC_Meta_Box_Product_Data::save`): core verifies `woocommerce_meta_nonce` and `edit_post` first, builds the object, runs `set_props()` and COGS, then fires **`woocommerce_admin_process_product_object`**, then calls `save()` once, then fires `woocommerce_process_product_meta_{type}( $post_id )`. The last hook runs **after** the save, so using it means a reload and a second save. Set your props on the object in `woocommerce_admin_process_product_object`.

```php
<?php
/**
 * Plugin Name:       My Plugin Gift Box
 * Requires at least: 7.0
 * Requires PHP:      8.4
 * Requires Plugins:  woocommerce
 * Text Domain:       my-plugin
 */

defined( 'ABSPATH' ) || exit;

add_action( 'before_woocommerce_init', static function () {
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
} );

add_action( 'woocommerce_init', 'my_plugin_load_gift_box_class' );
function my_plugin_load_gift_box_class() {
	if ( class_exists( 'My_Plugin_Product_Gift_Box' ) ) {
		return;
	}

	class My_Plugin_Product_Gift_Box extends WC_Product_Simple {
		public function get_type() {
			return 'gift_box';
		}

		/** @return int[] */
		public function get_item_ids( $context = 'view' ) {
			return array_map( 'absint', (array) $this->get_meta( '_my_plugin_item_ids', true, $context ) );
		}

		public function set_item_ids( array $ids ) {
			$this->update_meta_data( '_my_plugin_item_ids', array_values( array_filter( array_map( 'absint', $ids ) ) ) );
		}
	}
}

add_filter( 'woocommerce_product_class', 'my_plugin_gift_box_class', 10, 2 );
function my_plugin_gift_box_class( $classname, $product_type ) {
	return 'gift_box' === $product_type ? 'My_Plugin_Product_Gift_Box' : $classname;
}

add_filter( 'product_type_selector', 'my_plugin_gift_box_selector' );
function my_plugin_gift_box_selector( array $types ) {
	$types['gift_box'] = __( 'Gift box', 'my-plugin' );
	return $types;
}

add_filter( 'woocommerce_product_data_tabs', 'my_plugin_gift_box_tab' );
function my_plugin_gift_box_tab( array $tabs ) {
	$tabs['my_plugin_gift_box'] = array(
		'label'    => __( 'Gift box', 'my-plugin' ),
		'target'   => 'my_plugin_gift_box_data',
		'class'    => array( 'show_if_gift_box' ),
		'priority' => 25,
	);
	// Core tabs list the types they show for; add ours where it applies.
	$tabs['inventory']['class'][] = 'show_if_gift_box';
	return $tabs;
}

add_action( 'woocommerce_product_data_panels', 'my_plugin_gift_box_panel' );
function my_plugin_gift_box_panel() {
	global $product_object;
	$ids = $product_object instanceof My_Plugin_Product_Gift_Box ? $product_object->get_item_ids( 'edit' ) : array();
	echo '<div id="my_plugin_gift_box_data" class="panel woocommerce_options_panel hidden"><div class="options_group">';
	woocommerce_wp_text_input(
		array(
			'id'          => '_my_plugin_item_ids',
			'label'       => __( 'Products in the box', 'my-plugin' ),
			'description' => __( 'Comma-separated product IDs.', 'my-plugin' ),
			'desc_tip'    => true,
			'value'       => implode( ',', $ids ),
		)
	);
	echo '</div></div>';
}

// The pricing group and stock fields are marked show_if_simple/show_if_variable in core markup.
add_action( 'admin_enqueue_scripts', 'my_plugin_gift_box_admin_js', 20 );
function my_plugin_gift_box_admin_js() {
	$screen = get_current_screen();
	if ( ! $screen || 'product' !== $screen->id ) {
		return;
	}
	wp_add_inline_script(
		'wc-admin-product-meta-boxes',
		"jQuery( function ( $ ) {
			$( '.options_group.pricing, #inventory_product_data .show_if_simple' ).addClass( 'show_if_gift_box' );
			$( 'select#product-type' ).trigger( 'change' );
		} );"
	);
}

// Core already verified woocommerce_meta_nonce and edit_post before this hook.
add_action( 'woocommerce_admin_process_product_object', 'my_plugin_gift_box_save' );
function my_plugin_gift_box_save( $product ) {
	// phpcs:ignore WordPress.Security.NonceVerification.Missing
	if ( $product instanceof My_Plugin_Product_Gift_Box && isset( $_POST['_my_plugin_item_ids'] ) ) {
		// phpcs:ignore WordPress.Security.NonceVerification.Missing
		$product->set_item_ids( wp_parse_id_list( sanitize_text_field( wp_unslash( $_POST['_my_plugin_item_ids'] ) ) ) );
	}
}

// Single product form. The Add to Cart + Options block falls back to this action for custom types.
add_action( 'woocommerce_gift_box_add_to_cart', 'woocommerce_simple_add_to_cart', 30 );
```

### Blocks and Store API behaviour for custom types

- **Add to Cart + Options block:** core ships template parts only for simple, external, variable and grouped. For any other type it renders `do_action( 'woocommerce_{type}_add_to_cart' )`, i.e. the legacy PHP form, so the hook above is required in block themes too. That form posts the page (`?add-to-cart=`). It doesn't go through the Store API, and the POST bypasses the page cache. A block-native alternative is the **experimental** filter `__experimental_woocommerce_{type}_add_to_cart_with_options_block_template_part` (since 9.9.0; return the path to a block template part). It may change without notice.
- **Product Button block** (grids and single product): it renders the Interactivity API button that adds through the Store API when the product `supports( 'ajax_add_to_cart' )`, is purchasable, the AJAX add-to-cart option is on, and redirect-after-add is off. Extending `WC_Product_Simple` gives you that support.
- **Store API:** there is no product-type allowlist. `POST /wc/store/v1/cart/add-item` checks `is_purchasable()` (exists, viewable, price not empty; filter `woocommerce_is_purchasable`) and stock, runs `woocommerce_add_to_cart_validation`, then `woocommerce_store_api_validate_add_to_cart( $product, $request )` (throw `RouteException` to reject). The product schema exposes `type`, `is_purchasable`, `has_options`, `add_to_cart` and `brands`. Extra data goes through `woocommerce_store_api_register_endpoint_data` (see `references/cache-and-store-api.md`).

## Enums (`Automattic\WooCommerce\Enums\*`)

Public classes of string constants (`final class` + `const`, not native enums, because WooCommerce core still runs on PHP 7.4). They interoperate with every string API.

| Class | Values |
|---|---|
| `ProductType` | `SIMPLE`, `VARIABLE`, `GROUPED`, `EXTERNAL`, `VARIATION`. Custom types aren't included: use `wc_get_product_types()` for the filterable list. |
| `ProductStatus` | `PUBLISH`, `DRAFT`, `PENDING`, `PRIVATE`, `FUTURE`, `TRASH`, `AUTO_DRAFT` |
| `ProductStockStatus` | `IN_STOCK`, `OUT_OF_STOCK`, `ON_BACKORDER`, `LOW_STOCK` |
| `ProductTaxStatus` | `TAXABLE`, `SHIPPING`, `NONE` |
| `CatalogVisibility` | `VISIBLE`, `CATALOG`, `SEARCH`, `HIDDEN` |
| `OrderStatus` | Unprefixed: `PENDING`, `PROCESSING`, `ON_HOLD`, `COMPLETED`, `CANCELLED`, `REFUNDED`, `FAILED`, `CHECKOUT_DRAFT`, …, plus `PAYMENT_COMPLETE_STATUSES` |
| `OrderInternalStatus` | Prefixed: `wc-pending`, `wc-processing`, … |
| `OrderItemType` | `LINE_ITEM`, `FEE`, `SHIPPING`, `TAX`, `COUPON` |
| `PaymentGatewayFeature` | `PRODUCTS`, `REFUNDS`, `TOKENIZATION`, `ADD_PAYMENT_METHOD`, `SUBSCRIPTION_*`, … |
| Newer | `TaxBasedOn`, `DefaultCustomerAddress` (10.8); `CurrencyPosition`, `DimensionUnit`, `WeightUnit`, `TaxDisplayMode` (11.0); `CatalogSortOrder`, `StockDisplayFormat` (11.1) |

## Cost of Goods Sold

Feature `cost_of_goods_sold`: stable, **off by default**. When it's disabled, the COGS methods return `null`/`0` **and log `_doing_it_wrong`**, so guard every call:

```php
use Automattic\WooCommerce\Utilities\FeaturesUtil;

if ( FeaturesUtil::feature_is_enabled( 'cost_of_goods_sold' ) ) {
	$product->set_cogs_value( 1250.0 ); // 0.0 is stored as null ("no cost defined").
	$product->save();
	$cost = $product->get_cogs_total_value(); // float
}
```

Product: `set_cogs_value( ?float )`, `get_cogs_value(): ?float`, `get_cogs_effective_value()`, `get_cogs_total_value()`. Variations: `set_cogs_value_is_additive()`. Orders: `$order->has_cogs()`, `get_cogs_total_value()`, and per item `get_cogs_value()`. The controller class is Internal; the public check is `FeaturesUtil::feature_is_enabled()`.

## Brands

Brands are always on in core (since 9.6). The taxonomy is `product_brand` (hierarchical). Product props: `get_brand_ids()` / `set_brand_ids()`. Helpers: `wc_get_brands()`, `wc_get_brand_thumbnail_url()`, `wc_get_brand_thumbnail_image()`. REST `wc/v3/products/brands`; Store API `brands` on products. Query with `tax_query` (above).

## Variation gallery

Variation galleries are in core and always on since 11.1 (the `variation_gallery` feature id is deprecated, and checking it logs a deprecation). Use `$variation->get_gallery_image_ids()` / `set_gallery_image_ids()`. Data persists as `_product_image_gallery` on the variation.

## Product instance caching

`product_instance_caching` is **experimental**, off by default and declarable. When it's on, `wc_get_product()` returns a cached instance per request (non-persistent group), so two calls give you the same object. Don't mutate a product you don't intend to save, and don't rely on `wc_get_product()` to discard unsaved changes. It has no effect on full-page caching.

## Stock updates

Use `wc_update_product_stock( $product, $quantity = null, $operation = 'set', $updating = false )` with `'set'`, `'increase'` or `'decrease'`. It changes stock only if `$product->managing_stock()` is true, and it resolves the managing parent for variations. It writes through the data store, which avoids lost updates on concurrent orders, then saves the product: stock status, lookup tables, caches and `woocommerce_product_set_stock` / `woocommerce_variation_set_stock` all stay in sync. It returns the new quantity.

```php
$product_id = wc_get_product_id_by_sku( 'MY-SKU-750' );
$product    = $product_id ? wc_get_product( $product_id ) : null;
if ( $product && $product->managing_stock() && (int) $product->get_stock_quantity() !== 42 ) {
	wc_update_product_stock( $product, 42, 'set' );
}
```

Never `update_post_meta( $id, '_stock', … )`: stock status, the attribute lookup table, transients and hooks don't follow. Run bulk stock syncs in Action Scheduler (`references/action-scheduler.md`).

## Render-time cache traps (full-page HTML cache)

Product getters are session-free: `get_price()`, `get_regular_price()`, `get_stock_status()`, `get_meta()`, `get_brand_ids()`, `get_gallery_image_ids()`, `wc_price()`, `add_to_cart_url()`. They're safe in a cached template. The traps:

| API | What it reads | Rule |
|---|---|---|
| `$product->get_price_html()`, `wc_get_price_to_display()`, `wc_get_price_including_tax()` / `excluding_tax()` | `WC()->customer` VAT exemption and the tax location from the customer's taxable address | Cache-safe **only** if every visitor resolves to the same tax location: `woocommerce_default_customer_address` = shop base (or empty) and no geolocation. With taxes off it's moot. |
| Variable product price ranges | price hash includes tax rates and VAT exemption | Same rule. |
| Guest default location (`wc_get_customer_default_location()`) | geolocation when the option is `geolocation` or `geolocation_ajax` | Geolocation makes prices per visitor; `geolocation_ajax` adds a `?v=` redirect. Use **base**. |
| `WC()->cart`, `WC()->session`, `WC()->customer`, `wc_print_notices()`, `woocommerce_mini_cart()` | session | Never while rendering. Mini-Cart, Cart and Checkout blocks read the Store API in the browser. |
| `WC()->session->set()` during a GET render | may set the session cookie | Never. |

Never cache WC objects across requests yourself (transients holding `WC_Product`). Cache ids or scalars.

## Public vs internal APIs

- **Public:** `wc_*` functions, CRUD classes, `WC()`, `Automattic\WooCommerce\Utilities\*` (`FeaturesUtil`, `OrderUtil`, `LoggingUtil`, …), `Automattic\WooCommerce\Enums\*`, Store API functions (`woocommerce_store_api_register_endpoint_data`, …), documented hooks.
- **Internal:** anything under `Automattic\WooCommerce\Internal\*` or tagged `@internal For exclusive usage of WooCommerce core, backwards compatibility not guaranteed`. That includes `BatchProcessingController`, `ProductFeed`, `ShopperLists`, `OrderWithdrawal`, the COGS controller and `OrderNoteGroup`. They can change in any release.
- Hooks prefixed `__experimental_` are unstable by name.

## `wc_get_container()` limits

`wc_get_container()` returns WooCommerce's DI container. It only resolves classes in the `Automattic\WooCommerce` namespace and throws `ContainerException` for anything else, so **you cannot register your own services in it**. Fetching an `Internal\*` class from it works technically but carries no compatibility promise. Use your own container or plain objects for your plugin.

## Logging

```php
$logger = wc_get_logger();
$logger->error(
	sprintf( 'ERP rejected product %d: %s', $product_id, $message ),
	array( 'source' => 'my-plugin' )
);
```

- Levels: `emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info`, `debug` (or `log( $level, … )`). `source` groups entries under WooCommerce › Status › Logs.
- `LoggingUtil` (public) exposes `logging_is_enabled()`, `get_level_threshold()`, `get_retention_period()`, `get_logs_tab_url()`.
- **Never set `'remote-logging' => true` in the context.** That flag routes entries to WooCommerce's own remote logging service (core diagnostics, only with usage tracking enabled). It is not a channel for extensions, and third-party errors are dropped by default anyway.
- Never log secrets, full payment payloads or personal data.

## Common mistakes

- `wc_get_products( [ 'meta_query' => … ] )` and trusting the result. The meta query is silently dropped; use the data-store filter.
- Searching for a `brand` query var. Use `tax_query` on `product_brand`.
- Saving custom product fields in `woocommerce_process_product_meta_{type}` with `wc_get_product()` + `save()`. That's a second load and a second save; use `woocommerce_admin_process_product_object`.
- Extending `WC_Product` for a buyable type, which loses "Add to cart" text, URL and Store API add-to-cart in the Product Button block.
- Forgetting `woocommerce_{type}_add_to_cart`, so the single product page shows no form in block themes either.
- Treating the experimental Add to Cart + Options template-part filter as a stable API.
- Looking for a block-based product editor, `@woocommerce/product-editor` or `product_block_editor`. All removed in 11.0.
- Calling COGS methods without `FeaturesUtil::feature_is_enabled( 'cost_of_goods_sold' )`, which logs `_doing_it_wrong` and returns dummy values.
- Writing `_stock` / `_price` with `update_post_meta()` or SQL. Stock status, lookup tables and caches go stale.
- Replacing the `product` data store for a custom type instead of registering `product-{type}`; trying to replace the `order` store under HPOS.
- `get_price_html()` in a cached template while geolocation or "customer address = geolocate" is on.
- Registering your services in `wc_get_container()`, or depending on `Internal\*` classes.
- Adding `'remote-logging' => true` to log calls.
