# Orders on HPOS

How to read, write, query and extend orders in WooCommerce 11.x when orders live in the `wc_orders*` tables (HPOS). Everything goes through CRUD: `wc_get_order()`, `wc_get_orders()`, `$order->get_meta()`, `$order->update_meta_data()`, `$order->save()`. No posts-storage branches, no `'shop_order'` screen ids, no `get_post_meta()` on order ids.

Verified against the WooCommerce 11.2.1 source. File references are relative to the plugin root.

## HPOS status in 11.x

- HPOS is enabled automatically on new shops (no orders, no incompatible plugins). Posts storage still ships as "WordPress posts storage (legacy)" and compatibility mode (sync to posts) still exists. This skill assumes HPOS and never writes code for the legacy store.
- Under HPOS, `shop_order_placehold` posts exist only to reserve ids. Never read them.
- `hpos_datastore_caching` (off by default) caches raw order data in the data store. Whether it is on or off, never write to or flush the orders cache group yourself. Go through CRUD.

## Compatibility declarations

```php
add_action( 'before_woocommerce_init', static function () {
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'cart_checkout_blocks', __FILE__, true );
} );
```

- Signature: `FeaturesUtil::declare_compatibility( string $feature_id, string $plugin_file, bool $positive_compatibility = true ): bool`.
- **Timing:** it must run during `before_woocommerce_init`. Outside that hook it returns `false` and logs `_doing_it_wrong`.
- **Return value:** `true` when recorded, `false` for unknown feature ids or bad timing. Declaring a feature both compatible and incompatible throws an exception.
- `$plugin_file`: `__FILE__` from the main plugin file, or `'my-plugin/my-plugin.php'` from elsewhere.
- `custom_order_tables` is **mandatory**. It is the only feature whose default for undeclared plugins is *incompatible*, and an undeclared plugin blocks HPOS auto-enable.
- `cart_checkout_blocks` is tracked (default compatible). Declare it for anything that touches cart or checkout.
- Optional tracked ids (default compatible): `cost_of_goods_sold`, `product_instance_caching`, `fulfillments`, `order_withdrawal`, `customer_stock_notifications`. Declaring `false` shows warnings to the merchant.
- `product_block_editor` no longer exists. Declaring it returns `false` without any error. Delete it from old boilerplate.
- To test it, check that `FeaturesUtil::get_compatible_features_for_plugin( 'my-plugin/my-plugin.php' )['compatible']` contains `custom_order_tables` (only after `woocommerce_init`).

## `OrderUtil` (public: `Automattic\WooCommerce\Utilities\OrderUtil`)

The methods that are useful under HPOS-only code:

| Method | Use |
|---|---|
| `is_order_edit_screen( $type = 'shop_order' ): bool` | Enqueue admin assets only on the order edit screen (`current_screen` or later). |
| `is_order_list_table_screen( $type = 'shop_order' ): bool` | Same for the orders list. |
| `is_new_order_screen( $type = 'shop_order' ): bool` | "Add order" screen. |
| `get_order_admin_edit_url( int $order_id ): string` | Link to an order in admin (`admin.php?page=wc-orders&action=edit&id=…`). |
| `get_order_admin_new_url(): string` | "Add order" URL. |
| `get_order_admin_screen(): string` | Screen id. Throws outside admin, so prefer `wc_get_page_screen_id( 'shop-order' )`. |
| `is_order( $id, $types = [ 'shop_order' ] )` / `get_order_type( $id )` | Type checks without `get_post_type()`. |
| `get_table_for_orders()` / `get_table_for_order_meta()` | Table names, only for the rare unavoidable raw SQL (read-only reporting). |
| `get_count_for_type( $type )` | Order counts per status. |
| `remove_status_prefix( string $status ): string` | `wc-processing` → `processing`. |
| `custom_orders_table_datastore_cache_enabled()`, `orders_cache_usage_is_enabled()` | Diagnostics. |

Not for HPOS-only code: `custom_orders_table_usage_is_enabled()` as an if/else switch, `get_post_or_object_meta()`, `init_theorder_object()`, `get_post_or_order_id()`. They exist to support both stores. `wc_get_orders()` already uses the active store, and an `else` branch for posts storage only changes results silently.

## Order edit screen: meta box

The HPOS edit screen is `admin.php?page=wc-orders&action=edit&id=…`. In `src/Internal/Admin/Orders/Edit.php`:

- `add_meta_boxes` fires as `do_action( 'add_meta_boxes', $screen_id, $order )`, and so does `add_meta_boxes_{$screen_id}`.
- Meta boxes are rendered with `do_meta_boxes( $screen_id, $context, $order )`, so **the render callback receives a `WC_Order`**, not a `WP_Post`.
- `wc_get_page_screen_id( 'shop-order' )` returns `woocommerce_page_wc-orders` (or `admin_page_wc-orders` for users who can't see the WooCommerce menu). It only exists in admin, and it depends on the current user, so call it from admin hooks, not at file load.

### Saving: what core actually does (`Edit::handle_order_update`)

1. Returns unless `$_POST['action'] === 'edit_order'`.
2. `check_admin_referer()` on core's own nonce.
3. Saves taxonomies.
4. `do_action( 'woocommerce_process_shop_order_meta', $order_id, $order )`. `$order` is the screen's `WC_Order` instance. Core's own handlers on this hook are `WC_Meta_Box_Order_Items::save` (10), `WC_Meta_Box_Order_Downloads::save` (30), `WC_Meta_Box_Order_Data::save` (40) and `WC_Meta_Box_Order_Actions::save` (50).
5. `CustomMetaBox::handle_metadata_changes( $order )` saves the instance **only if the Custom Fields box changed**.
6. Redirect and `exit`.

**Core does not call `save()` on the instance it passes you.** `WC_Meta_Box_Order_Data::save` (priority 40) loads its own copy with `wc_get_order()` and saves that. So your handler must call `$order->save()` itself. On HPOS an update writes only changed props and meta (`get_changes()`), so saving the passed instance after priority 40 doesn't overwrite what core just saved. Priority 45 runs after core's data save and before `WC_Meta_Box_Order_Actions::save` (50), which may send emails from a fresh load.

Core has already checked its nonce when the hook fires, but add your own nonce and a capability check anyway: they tie the save to your box and make the handler safe if something else ever fires the hook.

## Orders list table on HPOS

All in `src/Internal/Admin/Orders/ListTable.php`. Use the order-type hooks, which don't need the screen id:

| Need | Hook |
|---|---|
| Add a column | filter `woocommerce_shop_order_list_table_columns` (`$columns`) |
| Render it | action `woocommerce_shop_order_list_table_custom_column` (`$column_name, WC_Order $order`) |
| Sortable columns | filter `woocommerce_shop_order_list_table_sortable_columns` |
| Filter dropdown UI | action `woocommerce_order_list_table_restrict_manage_orders` (`$order_type, $which`) |
| Apply the filter | filter `woocommerce_shop_order_list_table_prepare_items_query_args` (args for `wc_get_orders()`). The generic `woocommerce_order_list_table_prepare_items_query_args` runs first. |
| Bulk action entry | WP core filter `bulk_actions-{$screen_id}` |
| Bulk action handler | filter `handle_bulk_actions-{$screen_id}` (`$redirect_to, $action, int[] $ids`). Only called for actions core didn't handle. |

`manage_{$screen_id}_columns` and `manage_{$screen_id}_custom_column` also work, but they need the screen id at registration time.

Core handles any `mark_{status}` bulk action itself when `wc-{status}` is in `wc_get_order_statuses()`. A custom status only needs its entry added to `bulk_actions-{$screen_id}`.

## `wc_get_orders()`

`wc_get_orders( array $args )` returns `WC_Order[]`, ids (`'return' => 'ids'`), or a `stdClass` with `orders`, `total` and `max_num_pages` when `'paginate' => true`. Under HPOS the engine is `OrdersTableQuery`. Legacy keys are remapped (`posts_per_page` → `limit`, `post_status` → `status`, `post_type` → `type`, `paged` → `page`, `fields` → `return`, …).

| Arg | Notes |
|---|---|
| `status` | String or array. Prefix optional (`processing` = `wc-processing`). `'any'` = all valid statuses except `exclude_from_search` ones (e.g. `checkout-draft`). `'all'` = no status filter. Use `Automattic\WooCommerce\Enums\OrderStatus` constants. |
| `type` | Defaults to `wc_get_order_types( 'view-orders' )`. Pass `'shop_order'` to be explicit. |
| `limit` / `page` / `offset` | `limit` defaults to the `posts_per_page` option (10), **not** unlimited. `-1` = all. Batch big jobs instead. |
| `return` | `'objects'` (default) or `'ids'`. |
| `meta_query` | `WP_Query` syntax against `wc_orders_meta`: `relation`, `EXISTS`, `NOT EXISTS`, `LIKE`, `IN`, `BETWEEN`, `REGEXP`… Shortcuts `meta_key`/`meta_value`/`meta_compare` are merged in with AND. |
| `field_query` | Same syntax against order **columns**, with `field` instead of `key`: a prop name (`total`, `billing_country`, `payment_method`) or `table.column`. |
| `date_query` | `WP_Date_Query` syntax. Columns: `date_created_gmt`, `date_updated_gmt`, `date_paid_gmt`, `date_completed_gmt` (local names are mapped). Give `before`/`after` as full `Y-m-d H:i:s` UTC strings. |
| `date_created`, `date_paid`, `date_completed`, `date_modified` | Shorthand: `'2026-10-01'`, `'>=2026-10-01'`, `'2026-10-01...2026-10-31'`, or a timestamp (`'>' . $ts`). |
| `customer`, `billing_email`, `exclude`, `parent`, `orderby`, `order`, `no_found_rows` | As expected. |

**Date precision:** a `YYYY-MM-DD` value has day precision and is read as **store-local midnight**, converted to UTC (`OrdersTableQuery::local_time_to_gmt_date_query`). `a...b` includes all of day `b`. A unix timestamp is UTC with second precision. The 11.2 changelog entry on day-precision queries ("anchored at store-local midnight, `start...end` includes the full last day") is implemented in `WC_Data_Store_WP::parse_date_for_wp_query`, which serves `wc_get_products()` and the legacy store. HPOS `wc_get_orders()` already behaved this way. For "last N hours", pass a timestamp.

```php
use Automattic\WooCommerce\Enums\OrderStatus;

$order_ids = wc_get_orders(
	array(
		'type'        => 'shop_order',
		'status'      => array( OrderStatus::PROCESSING, OrderStatus::ON_HOLD ),
		'limit'       => 100,
		'return'      => 'ids',
		'meta_query'  => array(
			array(
				'key'     => '_my_plugin_exported',
				'compare' => 'NOT EXISTS',
			),
		),
		'field_query' => array(
			array(
				'field'   => 'total',
				'value'   => 50000,
				'compare' => '>=',
				'type'    => 'NUMERIC',
			),
		),
		'date_query'  => array(
			array(
				'column' => 'date_paid_gmt',
				'after'  => gmdate( 'Y-m-d H:i:s', time() - 30 * DAY_IN_SECONDS ),
			),
		),
	)
);
```

Extension points: `woocommerce_order_query_args` (args before the query), `woocommerce_order_query` (results), and `woocommerce_orders_table_query_clauses` (raw SQL clauses, last resort).

## Anti-patterns

| Don't | Do |
|---|---|
| `get_post_meta( $order_id, … )`, `update_post_meta()` | `$order->get_meta()`, `$order->update_meta_data()` + `$order->save()` |
| `new WP_Query( [ 'post_type' => 'shop_order' ] )`, `get_posts()` | `wc_get_orders()` |
| `$wpdb` on `posts`/`postmeta` for orders | `wc_get_orders()`. For reporting SQL, use `OrderUtil::get_table_for_orders()`. |
| `get_post( $order_id )`, `get_post_type( $order_id )` | `wc_get_order()`, `OrderUtil::get_order_type()` |
| `'shop_order'` as a screen id, `add_meta_box( …, 'shop_order' )` | `wc_get_page_screen_id( 'shop-order' )` |
| `if ( custom_orders_table_usage_is_enabled() ) … else …` | One unconditional CRUD path |
| `save()` several times in one flow | Buffer changes, save once |
| `woocommerce_data_stores` to replace `order` | Has no effect: HPOS swaps the store at `woocommerce_order_data_store` priority 999 |

## Custom order status

Both registrations are required. `wc_order_statuses` decides validity: `set_status()` with an unknown status silently falls back to `pending`. `woocommerce_register_shop_order_post_statuses` registers the status object that the list table and counts use (core runs it on `init` priority 9). The slug including `wc-` must be **≤ 20 characters**, because `wc_orders.status` is `varchar(20)`.

```php
add_filter( 'woocommerce_register_shop_order_post_statuses', 'my_plugin_register_shipped_status' );
function my_plugin_register_shipped_status( array $statuses ) {
	$statuses['wc-shipped'] = array(
		'label'                     => _x( 'Shipped', 'Order status', 'my-plugin' ),
		'public'                    => false,
		'exclude_from_search'       => false,
		'show_in_admin_all_list'    => true,
		'show_in_admin_status_list' => true,
		/* translators: %s: number of orders */
		'label_count'               => _n_noop( 'Shipped <span class="count">(%s)</span>', 'Shipped <span class="count">(%s)</span>', 'my-plugin' ),
	);
	return $statuses;
}

add_filter( 'wc_order_statuses', 'my_plugin_add_shipped_to_order_statuses' );
function my_plugin_add_shipped_to_order_statuses( array $statuses ) {
	$ordered = array();
	foreach ( $statuses as $key => $label ) {
		$ordered[ $key ] = $label;
		if ( 'wc-processing' === $key ) {
			$ordered['wc-shipped'] = _x( 'Shipped', 'Order status', 'my-plugin' );
		}
	}
	return $ordered;
}

// Make the transition email-capable: only actions listed here fire "{action}_notification".
add_filter( 'woocommerce_email_actions', 'my_plugin_shipped_email_actions' );
function my_plugin_shipped_email_actions( array $actions ) {
	$actions[] = 'woocommerce_order_status_processing_to_shipped';
	return $actions;
}

// Bulk action "Change status to shipped"; core handles mark_shipped itself.
add_action( 'admin_init', 'my_plugin_register_shipped_bulk_action' );
function my_plugin_register_shipped_bulk_action() {
	add_filter(
		'bulk_actions-' . wc_get_page_screen_id( 'shop-order' ),
		static function ( array $actions ) {
			$actions['mark_shipped'] = __( 'Change status to shipped', 'my-plugin' );
			return $actions;
		}
	);
}
```

Your email class then listens to `woocommerce_order_status_processing_to_shipped_notification` (see `references/emails.md`). If the status means "paid", add it to `woocommerce_order_is_paid_statuses` (the filter behind `wc_get_is_paid_statuses()`).

## Order lifecycle hooks: blocks flow vs classic-only

**Fire in the Checkout block / Store API flow:** `woocommerce_store_api_checkout_update_order_from_request( $order, $request )`, `woocommerce_checkout_validate_order_before_payment( $order, WP_Error $errors )`, `woocommerce_store_api_checkout_order_created`, `woocommerce_store_api_checkout_update_order_meta`, `woocommerce_store_api_checkout_order_processed( $order )`, the line-item hooks `woocommerce_checkout_create_order_line_item` / `_shipping_item` / `_fee_item` / `_tax_item` / `_coupon_item`, `woocommerce_new_order( $id, $order )` (on the draft-to-real transition, never for drafts), `woocommerce_update_order`, `woocommerce_order_status_{to}( $id, $order, $transition )`, `woocommerce_order_status_{from}_to_{to}( $id, $order )`, `woocommerce_order_status_changed( $id, $from, $to, $order )`, `woocommerce_pre_payment_complete`, `woocommerce_payment_complete( $id, $transaction_id )`, `woocommerce_thankyou` (Order Confirmation block).

**Classic-only (never fire in the Checkout block):** `woocommerce_checkout_process`, `woocommerce_before_checkout_process`, `woocommerce_after_checkout_validation`, `woocommerce_checkout_create_order`, `woocommerce_checkout_update_order_meta`, `woocommerce_checkout_order_created`, `woocommerce_checkout_order_processed`, `woocommerce_checkout_fields`.

### Do minimal work, enqueue Action Scheduler

Status and payment hooks run inside the shopper's Store API request, the gateway webhook or the admin save. A slow ERP/CRM/email call there delays checkout, can fail it, and repeats on retries. Inside the hook, check cheap conditions, then enqueue a unique job with ids only:

```php
add_action( 'woocommerce_order_status_changed', 'my_plugin_queue_order_sync', 10, 4 );
function my_plugin_queue_order_sync( $order_id, $from, $to, $order ) {
	if ( in_array( $to, wc_get_is_paid_statuses(), true ) ) {
		as_enqueue_async_action( 'my_plugin_sync_order', array( (int) $order_id, 1 ), 'my-plugin', true );
	}
}
```

The job, retries and backoff are covered in `references/action-scheduler.md`.

## Fulfillments (hidden, opt-in)

`fulfillments` ships in 11.x with `enabled_by_default => false` and `disable_ui => true`, so merchants can't turn it on from Settings › Features. Its option is `woocommerce_feature_fulfillments_enabled`. Code: `Automattic\WooCommerce\Admin\Features\Fulfillments` (a `Fulfillment extends WC_Data` with tracking number, provider, items), REST `wc/v3/orders/{id}/fulfillments`, hooks `woocommerce_fulfillment_after_create|after_update|after_delete`, providers via `woocommerce_fulfillment_shipping_providers`. Treat it as unstable: don't build production features on it yet. Until it ships, store tracking data in order meta as shown below.

## Order withdrawal (awareness)

`order_withdrawal` (stable since 11.1, off by default) adds a public My Account form for EU-style withdrawal requests. It is limited to a 14-day window, sets order meta `_order_withdrawal_requested = yes`, adds a note and sends two emails. It does not change the status or create refunds, and every API is `@internal`, so you can't extend it. Know it exists. Don't hook into it.

## Complete example: tracking number meta box + list column + WP-CLI report

```php
<?php
/**
 * Plugin Name:       My Plugin Tracking
 * Requires at least: 7.0
 * Requires PHP:      8.4
 * Requires Plugins:  woocommerce
 * Text Domain:       my-plugin
 */

use Automattic\WooCommerce\Enums\OrderStatus;

defined( 'ABSPATH' ) || exit;

const MY_PLUGIN_TRACKING_META = '_my_plugin_tracking_number';

add_action(
	'before_woocommerce_init',
	static function () {
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
	}
);

/* Meta box ------------------------------------------------------------- */

add_action( 'add_meta_boxes', 'my_plugin_add_tracking_meta_box', 10, 2 );
function my_plugin_add_tracking_meta_box( $screen_id, $order ) {
	if ( wc_get_page_screen_id( 'shop-order' ) !== $screen_id ) {
		return;
	}
	add_meta_box(
		'my-plugin-tracking',
		__( 'Tracking number', 'my-plugin' ),
		'my_plugin_render_tracking_meta_box',
		$screen_id,
		'side',
		'default'
	);
}

/**
 * @param WC_Order $order Passed by do_meta_boxes() on the HPOS edit screen.
 */
function my_plugin_render_tracking_meta_box( $order ) {
	if ( ! $order instanceof WC_Order ) {
		return;
	}
	wp_nonce_field( 'my_plugin_save_tracking', 'my_plugin_tracking_nonce' );
	printf(
		'<p><label for="my-plugin-tracking-number">%1$s</label><input type="text" class="widefat" id="my-plugin-tracking-number" name="my_plugin_tracking_number" value="%2$s" autocomplete="off" /></p>',
		esc_html__( 'Carrier tracking number', 'my-plugin' ),
		esc_attr( (string) $order->get_meta( MY_PLUGIN_TRACKING_META ) )
	);
}

// Priority 45: after WC_Meta_Box_Order_Data::save (40), before WC_Meta_Box_Order_Actions::save (50).
add_action( 'woocommerce_process_shop_order_meta', 'my_plugin_save_tracking_meta_box', 45, 2 );
function my_plugin_save_tracking_meta_box( $order_id, $order ) {
	if ( ! $order instanceof WC_Order ) {
		return;
	}
	if ( ! isset( $_POST['my_plugin_tracking_nonce'] )
		|| ! wp_verify_nonce( sanitize_key( wp_unslash( $_POST['my_plugin_tracking_nonce'] ) ), 'my_plugin_save_tracking' ) ) {
		return;
	}
	if ( ! current_user_can( 'edit_shop_orders' ) || ! isset( $_POST['my_plugin_tracking_number'] ) ) {
		return;
	}

	$tracking = sanitize_text_field( wp_unslash( $_POST['my_plugin_tracking_number'] ) );
	if ( (string) $order->get_meta( MY_PLUGIN_TRACKING_META ) === $tracking ) {
		return;
	}

	if ( '' === $tracking ) {
		$order->delete_meta_data( MY_PLUGIN_TRACKING_META );
	} else {
		$order->update_meta_data( MY_PLUGIN_TRACKING_META, $tracking );
	}
	$order->save(); // Core does not save the instance it passes to this hook.
}

/* Orders list column -------------------------------------------------- */

add_filter( 'woocommerce_shop_order_list_table_columns', 'my_plugin_tracking_column' );
function my_plugin_tracking_column( array $columns ) {
	$columns['my_plugin_tracking'] = esc_html__( 'Tracking', 'my-plugin' );
	return $columns;
}

add_action( 'woocommerce_shop_order_list_table_custom_column', 'my_plugin_render_tracking_column', 10, 2 );
function my_plugin_render_tracking_column( $column_name, $order ) {
	if ( 'my_plugin_tracking' !== $column_name || ! $order instanceof WC_Order ) {
		return;
	}
	$tracking = (string) $order->get_meta( MY_PLUGIN_TRACKING_META );
	echo '' === $tracking ? '&ndash;' : esc_html( $tracking );
}

/* WP-CLI: processing orders of the last 7 days without tracking -------- */

if ( defined( 'WP_CLI' ) && WP_CLI ) {
	WP_CLI::add_command( 'my-plugin missing-tracking', 'my_plugin_cli_missing_tracking' );
}

/**
 * Lists processing orders created in the last 7 days with no tracking number.
 *
 * ## OPTIONS
 *
 * [--format=<format>]
 * : table, csv, json, ids or count.
 * ---
 * default: table
 * ---
 *
 * ## EXAMPLES
 *
 *     wp my-plugin missing-tracking --format=csv
 *
 * @param array $args       Positional args (unused).
 * @param array $assoc_args Associative args.
 */
function my_plugin_cli_missing_tracking( $args, $assoc_args ) {
	$format = isset( $assoc_args['format'] ) ? $assoc_args['format'] : 'table';
	$page   = 1;
	$rows   = array();

	do {
		$orders = wc_get_orders(
			array(
				'type'         => 'shop_order',
				'status'       => array( OrderStatus::PROCESSING ),
				'date_created' => '>=' . ( time() - 7 * DAY_IN_SECONDS ), // UTC timestamp, second precision.
				'meta_query'   => array(
					'relation' => 'OR',
					array(
						'key'     => MY_PLUGIN_TRACKING_META,
						'compare' => 'NOT EXISTS',
					),
					array(
						'key'     => MY_PLUGIN_TRACKING_META,
						'value'   => '',
						'compare' => '=',
					),
				),
				'orderby'      => 'date',
				'order'        => 'ASC',
				'limit'        => 100,
				'page'         => $page,
			)
		);

		foreach ( $orders as $order ) {
			$created = $order->get_date_created();
			$rows[]  = array(
				'id'       => $order->get_id(),
				'number'   => $order->get_order_number(),
				'created'  => $created ? $created->date_i18n( 'Y-m-d H:i' ) : '',
				'customer' => $order->get_formatted_billing_full_name(),
				'total'    => $order->get_total(),
			);
		}
		++$page;
	} while ( count( $orders ) === 100 );

	if ( 'ids' === $format ) {
		WP_CLI::line( implode( ' ', wp_list_pluck( $rows, 'id' ) ) );
		return;
	}
	if ( 'count' === $format ) {
		WP_CLI::line( (string) count( $rows ) );
		return;
	}
	WP_CLI\Utils\format_items( $format, $rows, array( 'id', 'number', 'created', 'customer', 'total' ) );
}
```

The meta box save fires on HPOS only. Page-based pagination is safe here because the command doesn't change what it filters on.

## Common mistakes

- Registering the meta box on `'shop_order'` or `get_current_screen()->post_type`. Under HPOS the screen is `woocommerce_page_wc-orders`, so the box never appears.
- Treating the meta box callback argument as `WP_Post` (`$post->ID`). It is a `WC_Order`.
- Calling `update_meta_data()` in `woocommerce_process_shop_order_meta` and expecting core to persist it. Core saves a different instance, so call `$order->save()` yourself.
- Saving custom order fields on `save_post` or `save_post_shop_order`. Neither fires for HPOS orders.
- `wc_get_orders()` without `limit` and assuming you got everything. The default is 10 (`posts_per_page`).
- `'date_paid' => '2026-10-31'` and expecting UTC. Day-precision values are store-local days. Use a timestamp for exact UTC windows.
- A custom status slug like `wc-awaiting-shipment-x` (over 20 characters). It truncates in `wc_orders.status`.
- Registering a custom status in only one of the two filters, or forgetting `woocommerce_email_actions`, so its `_notification` hook never fires.
- Writing checkout logic in `woocommerce_checkout_order_processed` / `woocommerce_checkout_update_order_meta`. The Checkout block never fires them.
- Calling an external API inside `woocommerce_order_status_*` or `woocommerce_payment_complete`. Enqueue a unique Action Scheduler job.
- Calling `wc_get_page_screen_id()` at plugin load or on the front end. It's admin-only and user-dependent.
- Building on Fulfillments or Order withdrawal internals as if they were stable public APIs.
