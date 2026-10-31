# REST API, webhooks, Abilities and MCP (WooCommerce 11.2.1)

Server-to-server surfaces of WooCommerce: the back-office/integration REST API (`wc/v3`), your own admin routes, outgoing webhooks, and the Abilities that AI agents call through MCP.

**The split is strict:**

- **`wc/v3` = back office and integrations** (ERP, CRM, marketplaces, admin tools), authenticated with WC API keys or Application Passwords. Never called from the storefront.
- **Storefront = Store API (`wc/store/v1`)**, for everything a shopper's browser does: cart, checkout, storefront product data, and your own shopper endpoints registered as `wc/store/v1/my-plugin/…`. See [`cache-and-store-api.md`](cache-and-store-api.md) and `checkout-blocks.md`. Nothing in this file reaches the Checkout block.

The generic WordPress Abilities API (`wp_register_ability`, categories, `executeAbility` in JS) and Connectors/WP AI Client are covered by the sibling skill **`wp-block-editor`**. This file covers only the WooCommerce side.

## Which API for what

| Need | Use | Notes |
|---|---|---|
| Browser cart/checkout/product data, page-cache-safe | Store API `wc/store/v1` | `cache-and-store-api.md` |
| A custom endpoint the shopper's browser calls | `register_rest_route( 'wc/store/v1', '/my-plugin/…' )` | Store API namespace; never `admin-ajax`, `wc-ajax` or `my-plugin/v1` |
| ERP/CRM/back-office integration on WC objects | REST `wc/v3` | The current public version; API keys |
| Your plugin's own back-office resources | A controller added to `wc/v3`, `wc-my-plugin/v1`, or `my-plugin/v1` | See "Authentication" to choose |
| Push WC events to another system | WC webhooks (`WC_Webhook`) | Delivered from Action Scheduler |
| Let AI agents act on the store | Abilities + MCP Adapter | `meta.mcp.public` + strict permissions |

- **`wc/v4` is shipped but disabled.** It registers only when the wc-admin flag `rest-api-v4` is on, and that flag is `false` in the 11.2.1 build. Its controllers live under `src/Internal/` — do not build on them.
- **Legacy REST API (`/wc-api/v1..v3`) was removed in 9.0.** Core keeps a stub; the old behaviour needs the separate (unmaintained) *WooCommerce Legacy REST API* plugin, which HPOS flags as incompatible. Webhooks still set to `legacy_v3` throw "Unsupported webhook API version" without it.
- **The experimental GraphQL "Dual API" was removed from core in 11.2** (moved to a separate plugin; a no-op stub remains).
- **Lazy namespace loading (9.2+):** on a request to one WC namespace, the other *known* WC namespaces (`wc/v1..v4`, `wc-admin`, `wc-analytics`, `wc/store`, `wc/private`, `wc-telemetry`) are not loaded. Third-party namespaces always load. Filter: `wc_rest_should_load_namespace`.

## Authentication: why the namespace name matters

WooCommerce REST API keys (consumer key/secret) authenticate **only** routes whose namespace starts with `wc/` or `wc-`. Over HTTPS they go as HTTP Basic; over plain HTTP only OAuth 1.0a signatures are accepted. The key's permission (read / write / read_write) is then mapped to HTTP methods. Since 11.1 a WC key that authenticated is rejected (401) if WordPress resolves the request to a route outside those namespaces.

| Your namespace | WC API keys | Application Passwords / cookie+nonce |
|---|---|---|
| `my-plugin/v1` | No | Yes |
| `wc-my-plugin/v1` | Yes ("'wc-' lets third party plugins use our authentication methods") | Yes |
| `wc/v3` (controller added via filter) | Yes | Yes |

Application Passwords (WP core) work on every namespace and are the default choice for new integrations. Pick `wc-…` or `wc/v3` only when the client already holds WC keys. The filter `woocommerce_rest_is_request_to_rest_api` can widen WC's key check, but renaming the namespace is clearer.

## Adding a controller to `wc/v3`

Filter `woocommerce_rest_api_get_rest_namespaces` (since 4.5) maps namespace → `[ key => class ]`. WooCommerce instantiates each class with **no constructor arguments** on `rest_api_init` and calls `register_routes()`. Three traps:

1. `WC_REST_Controller` defaults to `protected $namespace = 'wc/v1'`. Set `'wc/v3'` yourself or the routes land in v1.
2. The class extends a WC class that only exists once WC's autoloader is registered. Require the file **inside** the filter callback, not at plugin load (plugins load alphabetically).
3. `batch_items()` lives on `WC_REST_Controller`, but `batch_items_permissions_check()` exists only on the posts/terms/reviews controllers. Implement it and register the `/batch` route yourself. Batch sub-requests call `update_item()` directly, **skipping route-argument validation**, so validate inside `update_item()`.

```php
// my-plugin.php — register the controller.
add_filter( 'woocommerce_rest_api_get_rest_namespaces', 'my_plugin_rest_namespaces' );

function my_plugin_rest_namespaces( array $namespaces ) {
	require_once __DIR__ . '/includes/class-my-plugin-rest-stock-controller.php';
	$namespaces['wc/v3']['my-plugin-stock'] = 'My_Plugin_REST_Stock_Controller'; // Key must be unique.
	return $namespaces;
}
```

```php
<?php
/**
 * GET|PUT /wc/v3/my-plugin-stock/{id} and POST /wc/v3/my-plugin-stock/batch.
 *
 * @package My_Plugin
 */

defined( 'ABSPATH' ) || exit;

/**
 * Warehouse stock per product, stored in product meta through CRUD.
 */
class My_Plugin_REST_Stock_Controller extends WC_REST_Controller {

	// Endpoint namespace. WC_REST_Controller defaults to wc/v1.
	protected $namespace = 'wc/v3';

	protected $rest_base = 'my-plugin-stock';

	public function register_routes() {
		register_rest_route(
			$this->namespace,
			'/' . $this->rest_base . '/(?P<id>[\d]+)',
			array(
				'args'   => array(
					'id' => array(
						'description' => __( 'Product ID.', 'my-plugin' ),
						'type'        => 'integer',
						'minimum'     => 1,
					),
				),
				array(
					'methods'             => WP_REST_Server::READABLE,
					'callback'            => array( $this, 'get_item' ),
					'permission_callback' => array( $this, 'get_item_permissions_check' ),
					'args'                => array( 'context' => $this->get_context_param( array( 'default' => 'view' ) ) ),
				),
				array(
					'methods'             => WP_REST_Server::EDITABLE,
					'callback'            => array( $this, 'update_item' ),
					'permission_callback' => array( $this, 'update_item_permissions_check' ),
					'args'                => $this->get_endpoint_args_for_item_schema( WP_REST_Server::EDITABLE ),
				),
				'schema' => array( $this, 'get_public_item_schema' ),
			)
		);

		register_rest_route(
			$this->namespace,
			'/' . $this->rest_base . '/batch',
			array(
				array(
					'methods'             => WP_REST_Server::EDITABLE,
					'callback'            => array( $this, 'batch_items' ), // Inherited; limit 100 items (filter woocommerce_rest_batch_items_limit), 413 above it.
					'permission_callback' => array( $this, 'batch_items_permissions_check' ),
					'args'                => $this->get_endpoint_args_for_item_schema( WP_REST_Server::EDITABLE ),
				),
				'schema' => array( $this, 'get_public_batch_schema' ),
			)
		);
	}

	// Object-level read check (read_private_products), filterable via woocommerce_rest_check_permissions.
	public function get_item_permissions_check( $request ) {
		if ( ! wc_rest_check_post_permissions( 'product', 'read', (int) $request['id'] ) ) {
			return new WP_Error( 'woocommerce_rest_cannot_view', __( 'Sorry, you cannot view this resource.', 'my-plugin' ), array( 'status' => rest_authorization_required_code() ) );
		}
		return true;
	}

	// Object-level edit check (edit_post on the product).
	public function update_item_permissions_check( $request ) {
		if ( ! wc_rest_check_post_permissions( 'product', 'edit', (int) $request['id'] ) ) {
			return new WP_Error( 'woocommerce_rest_cannot_edit', __( 'Sorry, you are not allowed to edit this resource.', 'my-plugin' ), array( 'status' => rest_authorization_required_code() ) );
		}
		return true;
	}

	// Batch check. batch_items() also runs update_item_permissions_check() per item.
	public function batch_items_permissions_check( $request ) {
		if ( ! wc_rest_check_post_permissions( 'product', 'batch' ) ) {
			return new WP_Error( 'woocommerce_rest_cannot_batch', __( 'Sorry, you are not allowed to batch manipulate this resource.', 'my-plugin' ), array( 'status' => rest_authorization_required_code() ) );
		}
		return true;
	}

	public function get_item( $request ) {
		$product = wc_get_product( (int) $request['id'] );
		if ( ! $product ) {
			return new WP_Error( 'my_plugin_rest_invalid_id', __( 'Invalid product ID.', 'my-plugin' ), array( 'status' => 404 ) );
		}
		return $this->prepare_item_for_response( $product, $request );
	}

	// PUT one item (also called directly by batch_items(), without route validation).
	public function update_item( $request ) {
		$product = wc_get_product( (int) $request['id'] );
		if ( ! $product ) {
			return new WP_Error( 'my_plugin_rest_invalid_id', __( 'Invalid product ID.', 'my-plugin' ), array( 'status' => 404 ) );
		}

		if ( isset( $request['warehouse_stock'] ) ) {
			$schema = $this->get_item_schema();
			$valid  = rest_validate_value_from_schema( $request['warehouse_stock'], $schema['properties']['warehouse_stock'], 'warehouse_stock' );
			if ( is_wp_error( $valid ) ) {
				return $valid;
			}
			$product->update_meta_data( '_my_plugin_warehouse_stock', (int) $request['warehouse_stock'] );
			$product->save();
		}

		return $this->prepare_item_for_response( $product, $request );
	}

	public function prepare_item_for_response( $product, $request ) {
		$data = array(
			'id'              => $product->get_id(),
			'warehouse_stock' => (int) $product->get_meta( '_my_plugin_warehouse_stock' ),
		);

		$context  = ! empty( $request['context'] ) ? $request['context'] : 'view';
		$data     = $this->add_additional_fields_to_object( $data, $request );
		$data     = $this->filter_response_by_context( $data, $context );
		$response = rest_ensure_response( $data );
		$response->add_link( 'up', rest_url( 'wc/v3/products/' . $product->get_id() ) );

		return $response;
	}

	// JSON schema. add_additional_fields_schema() applies register_rest_field() fields and the woocommerce_rest_{title}_schema filter.
	public function get_item_schema() {
		$schema = array(
			'$schema'    => 'http://json-schema.org/draft-04/schema#',
			'title'      => 'my_plugin_stock',
			'type'       => 'object',
			'properties' => array(
				'id'              => array(
					'description' => __( 'Product ID.', 'my-plugin' ),
					'type'        => 'integer',
					'context'     => array( 'view', 'edit' ),
					'readonly'    => true,
				),
				'warehouse_stock' => array(
					'description' => __( 'Units in the external warehouse.', 'my-plugin' ),
					'type'        => 'integer',
					'minimum'     => 0,
					'context'     => array( 'view', 'edit' ),
				),
			),
		);

		return $this->add_additional_fields_schema( $schema );
	}
}
```

Batch body: `{ "update": [ { "id": 12, "warehouse_stock": 40 } ] }`. Unsupported `create`/`delete` entries come back as per-item errors (the inherited `WP_REST_Controller` methods return "not implemented").

Extend `WC_REST_CRUD_Controller` only if you model a real `WC_Data` entity with its own data store; most extensions want `WC_REST_Controller`. `RestApiControllerBase` and the `RestApiCache` trait are in `src/Internal/` — no BC promise.

### Permission helpers

All pass through `woocommerce_rest_check_permissions( $permission, $context, $object_id, $object_type )`.

| Helper | Maps to |
|---|---|
| `wc_rest_check_post_permissions( $post_type, $context, $object_id )` | `read`→`read_private_posts`, `create`→`publish_posts`, `edit`→`edit_post`, `delete`→`delete_post`, `batch`→`edit_others_posts`, on that post type's caps. Works for `shop_order` with HPOS (the post type stays registered); WC's own order abilities use `( 'shop_order', 'edit', $id )`. |
| `wc_rest_check_user_permissions( $context, $object_id )` | `list_users` / `create_customers` / `edit_users` / `delete_users` / `promote_users`; shop managers only edit roles in `woocommerce_shop_manager_editable_roles`. |
| `wc_rest_check_product_term_permissions( $taxonomy, $context, $object_id )` | Taxonomy caps. |
| `wc_rest_check_manager_permissions( $object, $context )` | **Only 7 keys:** `reports`, `settings`, `system_status`, `attributes`, `shipping_methods`, `payment_gateways`, `webhooks`. Any other key → undefined-index warning and `false`. For your own objects call `current_user_can( 'manage_woocommerce' )`. |

## Standalone routes for your own plugin

**Shopper-facing routes do not belong here.** Register them as `wc/store/v1/my-plugin/…` (details in `cache-and-store-api.md`; a tested example is in `testing.md`). Verified in 11.2.1: a route under `/wc/store/` gets the Store API CORS headers (`Nonce`, `Cart-Token`), the Cart-Token session handler and the Store API rate limiter when that is enabled — but **not** the nonce check or cart loading, which live in core's cart route classes. A custom write route must verify the `Nonce` header (`wp_verify_nonce( $nonce, 'wc_store_api' )`) and call `wc_load_cart()` before touching `WC()->cart`.

For back-office routes where WC keys are not needed, register a normal WP route. Same rules as any REST route: schema-typed args, a real `permission_callback`, CRUD for WC data.

```php
add_action( 'rest_api_init', 'my_plugin_register_routes' );

function my_plugin_register_routes() {
	register_rest_route(
		'my-plugin/v1',
		'/orders/(?P<id>[\d]+)/sync-status',
		array(
			'methods'             => WP_REST_Server::READABLE,
			'callback'            => 'my_plugin_rest_sync_status',
			'permission_callback' => static function ( WP_REST_Request $request ) {
				return wc_rest_check_post_permissions( 'shop_order', 'read', (int) $request['id'] );
			},
			'args'                => array(
				// Never 'validate_callback' => 'is_numeric': WP passes 3 args and PHP 8 throws ArgumentCountError.
				'id' => array(
					'type'     => 'integer',
					'minimum'  => 1,
					'required' => true,
				),
			),
		)
	);
}

function my_plugin_rest_sync_status( WP_REST_Request $request ) {
	$order = wc_get_order( (int) $request['id'] );
	if ( ! $order ) {
		return new WP_Error( 'my_plugin_not_found', __( 'Order not found.', 'my-plugin' ), array( 'status' => 404 ) );
	}
	return rest_ensure_response(
		array(
			'order_id'  => $order->get_id(),
			'synced_at' => $order->get_meta( '_my_plugin_synced_at' ) ? $order->get_meta( '_my_plugin_synced_at' ) : null,
		)
	);
}
```

Inbound webhooks from third parties (payment providers, ERPs) are REST routes too: verify an HMAC of the raw body with `hash_equals`, dedupe by event id, enqueue a unique Action Scheduler job, answer 200 fast. Full pattern in `payments.md`.

## Extending core responses (products, orders, coupons, customers)

| Hook | When |
|---|---|
| `woocommerce_rest_prepare_{post_type}_object( $response, $object, $request )` | Response for `product`, `product_variation`, `shop_order`, `shop_coupon`. Customers: `woocommerce_rest_prepare_customer` (receives `WP_User`). |
| `woocommerce_rest_pre_insert_{post_type}_object( $object, $request, $creating )` | Before the controller's single `save()` — the cheapest place to write. |
| `woocommerce_rest_insert_{post_type}_object( $object, $request, $creating )` | After save (action). |
| `woocommerce_rest_{post_type}_object_query( $args, $request )` | Collection query args. |
| `woocommerce_rest_{object_type}_schema( $properties )` | Schema properties. |

**`register_rest_field()` is the preferred way to add a field** (it adds schema, get and update). The object type is the schema title: `'product'`, `'shop_order'`, `'shop_coupon'`. In the CRUD controllers `create_item()`/`update_item()` call `save_object()` **first** and only then `update_additional_fields_for_object()`, so **your `update_callback` must call `$object->save()` itself**, or the value is lost.

```php
add_action( 'rest_api_init', 'my_plugin_register_order_fields' );

function my_plugin_register_order_fields() {
	register_rest_field(
		'shop_order',
		'my_plugin_delivery_date',
		array(
			'get_callback'    => 'my_plugin_rest_get_delivery_date',
			'update_callback' => 'my_plugin_rest_update_delivery_date',
			'schema'          => array(
				'description' => __( 'Requested delivery date (YYYY-MM-DD).', 'my-plugin' ),
				'type'        => array( 'string', 'null' ),
				'pattern'     => '^\d{4}-\d{2}-\d{2}$',
				'context'     => array( 'view', 'edit' ),
			),
		)
	);
}

/**
 * The get_callback receives the prepared response array, not the object: load it through CRUD.
 *
 * @param array $data Prepared order data.
 * @return string|null
 */
function my_plugin_rest_get_delivery_date( $data ) {
	$order = wc_get_order( $data['id'] );
	if ( ! $order ) {
		return null;
	}
	$date = $order->get_meta( '_my_plugin_delivery_date' );
	return $date ? $date : null;
}

/**
 * Runs AFTER the controller saved the order — save again or the change is lost.
 *
 * @param string|null $value Value from the request.
 * @param WC_Order    $order Order already saved by the controller.
 * @return true
 */
function my_plugin_rest_update_delivery_date( $value, $order ) {
	if ( null === $value || '' === $value ) {
		$order->delete_meta_data( '_my_plugin_delivery_date' );
	} else {
		$order->update_meta_data( '_my_plugin_delivery_date', sanitize_text_field( $value ) );
	}
	$order->save();
	return true;
}
```

To avoid the second save, keep the field read-only in `register_rest_field()` (schema + `get_callback`) and write it in `woocommerce_rest_pre_insert_shop_order_object`, reading `$request['my_plugin_delivery_date']`.

Read-only computed data with the object already loaded:

```php
add_filter( 'woocommerce_rest_prepare_shop_order_object', 'my_plugin_rest_prepare_order', 10, 3 );

function my_plugin_rest_prepare_order( $response, $order, $request ) {
	$data                      = $response->get_data();
	$data['my_plugin_bottles'] = $order->get_item_count(); // Sum of line-item quantities.
	$response->set_data( $data );
	return $response;
}
```

Side effects to know:

- Webhook payloads for core resources are these same REST responses, so your fields **also appear in webhooks** — never add secrets or PII you would not send out.
- `rest_api_caching` (experimental since 10.5, off by default, needs a persistent object cache) caches Products v2/v3 responses and invalidates them when product data changes — **not** when your external data (ERP stock, live prices) changes. Bump your own version or document `?_skip_cache=1`.
- None of this reaches the Store API or the Checkout block. For those use `woocommerce_store_api_register_endpoint_data()` (`cache-and-store-api.md`).

## Webhooks

`WC_Webhook` (`includes/class-wc-webhook.php`):

- Matching hooks are collected during the request and processed on `shutdown`. Delivery is **async by default** (`woocommerce_webhook_deliver_async`, default true): an Action Scheduler action `woocommerce_deliver_webhook_async` in group `woocommerce-webhooks`, skipped if the same webhook+arg is already pending within the next 10 minutes.
- Core payload = the REST response of `/wc/{v1|v2|v3}/{resource}s/{id}`, built **as the webhook's owner user**. Valid API versions: `wp_api_v1`, `wp_api_v2`, `wp_api_v3`.
- Headers: `X-WC-Webhook-Source|Topic|Resource|Event|Signature|ID|Delivery-ID`. Signature = base64 HMAC-SHA256 (`woocommerce_webhook_hash_algorithm`) of the JSON body with the secret.
- Delivery uses `wp_safe_remote_request()`: localhost and private IPs are refused. More than 5 consecutive failures (`woocommerce_max_webhook_delivery_failures`) disable the webhook (`woocommerce_webhook_disabled_due_delivery_failures`).
- Other filters: `woocommerce_webhook_should_deliver`, `woocommerce_webhook_http_args`, `woocommerce_webhook_payload`, `woocommerce_webhook_enable_delivery_log` (11.1).

**Topics.** Default resources `coupon`, `customer`, `order`, `product`; events `created`, `updated`, `deleted`, `restored`, `published`. Since **10.9** `WC_Webhook::get_default_topic_hooks()` (public static) is the source of truth and validation **rejects default pairs with no hooks** (e.g. `order.published`, `customer.restored`). `WC_Webhook::set_topic()` silently stores `''` for an invalid topic.

**Action topics** `action.{hook}` need no registration when the hook starts with `woocommerce_` or `wc_`; the payload is `{ "action": hook, "arg": first hook argument }`. Pass an ID as the first argument, never an object full of PII.

**Custom resource topic** (e.g. `subscription.renewed`). Add the filters at plugin load: webhooks are loaded and hooked during `WooCommerce::init()`.

```php
add_filter( 'woocommerce_valid_webhook_resources', 'my_plugin_webhook_resources' );
add_filter( 'woocommerce_valid_webhook_events', 'my_plugin_webhook_events' );
add_filter( 'woocommerce_webhook_topic_hooks', 'my_plugin_webhook_topic_hooks' );
add_filter( 'woocommerce_webhook_topics', 'my_plugin_webhook_topics' );
add_filter( 'woocommerce_webhook_payload', 'my_plugin_webhook_payload', 10, 4 );

function my_plugin_webhook_resources( $resources ) {
	$resources[] = 'subscription';
	return $resources;
}

function my_plugin_webhook_events( $events ) {
	$events[] = 'renewed';
	return $events;
}

// Topic → hooks that trigger it. The hook's first argument becomes $resource_id.
function my_plugin_webhook_topic_hooks( $topic_hooks ) {
	$topic_hooks['subscription.renewed'] = array( 'my_plugin_subscription_renewed' );
	return $topic_hooks;
}

// Admin dropdown (WooCommerce → Settings → Advanced → Webhooks).
function my_plugin_webhook_topics( $topics ) {
	$topics['subscription.renewed'] = __( 'Subscription renewed', 'my-plugin' );
	return $topics;
}

// Non-core resources get an empty payload unless you build it here.
function my_plugin_webhook_payload( $payload, $resource, $resource_id, $webhook_id ) {
	if ( 'subscription' !== $resource ) {
		return $payload;
	}
	return my_plugin_subscription_to_array( (int) $resource_id ); // Your own serializer.
}

// Somewhere in your code, with an ID as the only argument:
// do_action( 'my_plugin_subscription_renewed', $subscription_id );
```

Creating one programmatically (after those filters exist):

```php
$webhook = new WC_Webhook();
$webhook->set_name( 'ERP: subscription renewed' );
$webhook->set_user_id( get_current_user_id() ); // Payload is built with this user's permissions.
$webhook->set_topic( 'subscription.renewed' );  // Invalid topic → stored as ''.
$webhook->set_delivery_url( 'https://erp.example.com/hooks/woo' );
$webhook->set_secret( wp_generate_password( 50, true, true ) );
$webhook->set_api_version( 'wp_api_v3' );
$webhook->set_status( 'active' );
$webhook->save();
```

Receiver check (any PHP endpoint): `hash_equals( $header_signature, base64_encode( hash_hmac( 'sha256', $raw_body, $secret, true ) ) )` on the **raw** body, before decoding JSON.

## WooCommerce Abilities

Always on in 11.2.1 (no flag). WooCommerce registers categories `woocommerce` and `woocommerce-rest` on `wp_abilities_api_categories_init`, and its abilities on `wp_abilities_api_init` (plus the pre-6.9 `abilities_api_init`).

Seven core domain abilities (since 10.9), category `woocommerce`, `meta.show_in_rest = true`, `meta.mcp = { public: true, type: 'tool' }`:

| Ability | readonly | destructive | Permission |
|---|---|---|---|
| `woocommerce/orders-query` | yes | no | order query check |
| `woocommerce/order-add-note` | no | no | `wc_rest_check_post_permissions( 'shop_order', 'edit', $id )` |
| `woocommerce/order-update-status` | no | yes | same |
| `woocommerce/products-query` | yes | no | `wc_rest_check_post_permissions( 'product', 'read', $id )` |
| `woocommerce/product-create` | no | no | product create check |
| `woocommerce/product-update` | no | yes | product update check |
| `woocommerce/product-delete` | no | yes | product delete check |

There are also REST-derived abilities (`woocommerce/products-list|get|create|update|delete`, `woocommerce/orders-list|get|create|update`, category `woocommerce-rest`) registered **only during a request to the deprecated `/woocommerce/mcp` endpoint**. They are removed in 11.3; never reference them.

**The `woocommerce/` prefix is reserved.** Non-core definitions using it are silently skipped, and WooCommerce unregisters any earlier registration of a core name and replaces it (logged to source `woocommerce-abilities`). Use your own prefix: `my-plugin/…`.

### Option A — plain WordPress API (portable, recommended)

```php
add_action( 'wp_abilities_api_init', 'my_plugin_register_abilities' ); // Registering elsewhere → _doing_it_wrong.

function my_plugin_register_abilities() {
	wp_register_ability(
		'my-plugin/stock-forecast',
		array(
			'label'               => __( 'Stock forecast', 'my-plugin' ),
			'description'         => __( 'Estimate how many days until a product runs out of stock.', 'my-plugin' ),
			'category'            => 'woocommerce', // Registered by WooCommerce.
			'input_schema'        => array(
				'type'                 => 'object',
				'properties'           => array(
					'product_id' => array(
						'type'    => 'integer',
						'minimum' => 1,
					),
				),
				'required'             => array( 'product_id' ),
				'additionalProperties' => false,
			),
			'output_schema'       => array(
				'type'       => 'object',
				'properties' => array(
					'product_id' => array( 'type' => 'integer' ),
					'stock'      => array( 'type' => 'integer' ),
					'days_left'  => array( 'type' => array( 'integer', 'null' ) ),
				),
			),
			'execute_callback'    => 'my_plugin_ability_stock_forecast',
			'permission_callback' => static function ( $input ) {
				$product_id = isset( $input['product_id'] ) ? absint( $input['product_id'] ) : 0;
				return $product_id > 0 && wc_rest_check_post_permissions( 'product', 'read', $product_id );
			},
			'meta'                => array(
				'show_in_rest' => true,                                         // /wp-abilities/v1.
				'mcp'          => array( 'public' => true, 'type' => 'tool' ), // MCP Adapter default server.
				'annotations'  => array(
					'readonly'    => true,
					'destructive' => false,
					'idempotent'  => true,
				),
			),
		)
	);
}

function my_plugin_ability_stock_forecast( $input ) {
	$product = wc_get_product( (int) $input['product_id'] );
	if ( ! $product || ! $product->managing_stock() ) {
		return new WP_Error( 'my_plugin_no_stock', __( 'Product not found or stock not managed.', 'my-plugin' ) );
	}
	$per_day = (float) $product->get_meta( '_my_plugin_units_per_day' );
	$stock   = (int) $product->get_stock_quantity();

	return array(
		'product_id' => $product->get_id(),
		'stock'      => $stock,
		'days_left'  => $per_day > 0 ? (int) floor( $stock / $per_day ) : null,
	);
}
```

### Option B — WooCommerce definition class (10.9+)

Implement the public interface `Automattic\WooCommerce\Abilities\AbilityDefinition` (`static get_name(): string`, `static get_registration_args(): array`) and append the class name to `woocommerce_ability_definition_classes`; WooCommerce registers it on `wp_abilities_api_init`. Returning a subset from the filter never removes core abilities. Do not extend `AbstractDomainAbility` or its traits (`src/Internal/`). Requires `WC requires at least: 10.9`.

```php
add_filter( 'woocommerce_ability_definition_classes', 'my_plugin_ability_classes' );

function my_plugin_ability_classes( $classes ) {
	require_once __DIR__ . '/includes/class-my-plugin-ability-set-delivery-date.php'; // Interface exists only once WC loaded.
	$classes[] = 'My_Plugin_Ability_Set_Delivery_Date';
	return $classes;
}
```

```php
<?php
/**
 * Ability: my-plugin/order-set-delivery-date.
 *
 * @package My_Plugin
 */

defined( 'ABSPATH' ) || exit;

use Automattic\WooCommerce\Abilities\AbilityDefinition;

/**
 * Sets the requested delivery date on an order.
 */
final class My_Plugin_Ability_Set_Delivery_Date implements AbilityDefinition {

	public static function get_name(): string {
		return 'my-plugin/order-set-delivery-date';
	}

	public static function get_registration_args(): array {
		return array(
			'label'               => __( 'Set order delivery date', 'my-plugin' ),
			'description'         => __( 'Store the requested delivery date (YYYY-MM-DD) on an order.', 'my-plugin' ),
			'category'            => 'woocommerce',
			'input_schema'        => array(
				'type'                 => 'object',
				'properties'           => array(
					'order_id' => array(
						'type'    => 'integer',
						'minimum' => 1,
					),
					'date'     => array(
						'type'    => 'string',
						'pattern' => '^\d{4}-\d{2}-\d{2}$',
					),
				),
				'required'             => array( 'order_id', 'date' ),
				'additionalProperties' => false,
			),
			'output_schema'       => array(
				'type'       => 'object',
				'properties' => array(
					'order_id'      => array( 'type' => 'integer' ),
					'delivery_date' => array( 'type' => 'string' ),
				),
			),
			'execute_callback'    => array( __CLASS__, 'execute' ),
			'permission_callback' => array( __CLASS__, 'can_edit_order' ),
			'meta'                => array(
				'show_in_rest' => true,
				'mcp'          => array( 'public' => true, 'type' => 'tool' ),
				'annotations'  => array(
					'readonly'    => false,
					'destructive' => false,
					'idempotent'  => true,
				),
			),
		);
	}

	// Object-level check, the same one WooCommerce uses for its order abilities.
	public static function can_edit_order( $input = array() ): bool {
		$order_id = is_array( $input ) && isset( $input['order_id'] ) ? absint( $input['order_id'] ) : 0;
		return $order_id > 0 && wc_rest_check_post_permissions( 'shop_order', 'edit', $order_id );
	}

	public static function execute( $input ) {
		$order = wc_get_order( (int) $input['order_id'] );
		if ( ! $order ) {
			return new WP_Error( 'my_plugin_not_found', __( 'Order not found.', 'my-plugin' ) );
		}
		$order->update_meta_data( '_my_plugin_delivery_date', sanitize_text_field( $input['date'] ) );
		$order->save();

		return array(
			'order_id'      => $order->get_id(),
			'delivery_date' => $order->get_meta( '_my_plugin_delivery_date' ),
		);
	}
}
```

**Execution order** in `WP_Ability::execute()`: normalize input → validate against `input_schema` → `permission_callback` → `execute_callback` → validate against `output_schema`. A permission failure returns `WP_Error( 'ability_invalid_permissions' )`. Test with `wp_get_ability( 'my-plugin/stock-forecast' )->execute( array( 'product_id' => $id ) )` after `wp_set_current_user()` (see `testing.md`).

11.3 adds experimental "extensible abilities" (`ExtensibleAbilities::register_field()` etc., fields under `extensions.<namespace>`); not in 11.2.1 and still changing — see `releases.md`.

## MCP

**11.2.1 (installed):**

- Feature `mcp_integration` ("WooCommerce MCP"), experimental, off by default; toggle in WooCommerce → Settings → Advanced → Features (option `woocommerce_feature_mcp_integration_enabled`). Requires non-plain permalinks.
- WooCommerce bundles `wordpress/mcp-adapter` 0.3.0 and `wordpress/abilities-api` 0.4.0 and boots the adapter on `rest_api_init` only when the flag is on. The adapter's default server `/wp-json/mcp/mcp-adapter-default-server` then exposes abilities with `meta.mcp.public = true`.
- WooCommerce also creates `/wp-json/woocommerce/mcp`, described in code as the **deprecated** endpoint: header `X-MCP-API-Key: consumer_key:consumer_secret`, HTTPS required, tools limited to abilities flagged `expose_in_deprecated_woocommerce_mcp`.

**11.3 (post-11.2.1, release planned week of 2026-11-09):** `/wp-json/woocommerce/mcp` returns 410, `X-MCP-API-Key`, the `mcp_integration` flag, the REST-derived abilities and their filters are gone, and the adapter is no longer bundled. The path becomes the **MCP Adapter plugin** (wp.org `mcp-adapter`) with its default server, authenticated as a WordPress user with an **Application Password** over HTTPS (or STDIO via WP-CLI locally).

**What to build on (works on both):**

- Register abilities with your own prefix, `meta.mcp.public = true` and `meta.mcp.type = 'tool'`.
- Strict, object-level `permission_callback`s — an agent acts with the capabilities of the WordPress user it authenticates as. Recommend a dedicated low-privilege user: order abilities expose customer PII.
- Accurate `annotations` (`readonly`, `destructive`, `idempotent`) and tight `input_schema` with `additionalProperties: false`.
- Never build on `/woocommerce/mcp`, `X-MCP-API-Key`, `expose_in_deprecated_woocommerce_mcp` or the `woocommerce_mcp_*` filters.
- If you depend on the adapter at runtime, guard with `class_exists( 'WP\MCP\Core\McpAdapter' )`. Running the wp.org adapter plugin next to 11.2's bundled copy is untested — upgrade to 11.3 first.

## Blueprint (settings export/import)

Feature `blueprint`, labelled beta but on by default. Exports/imports WooCommerce settings as a JSON schema: WP-CLI `wp wc blueprint export <file> [--steps=a,b]` and `wp wc blueprint import <file> [--yes]`; REST under `wc-admin/blueprint/*`. Extensions add steps through `wooblueprint_exporters` / `wooblueprint_importers` with classes implementing `Automattic\WooCommerce\Blueprint\Exporters\StepExporter` (`export()`, `get_step_name()`, `check_step_capabilities()`).

## Common mistakes

- Calling `wc/v3` from the storefront, or putting a shopper endpoint on `my-plugin/v1`. Storefront traffic goes to the Store API; custom shopper routes are `wc/store/v1/my-plugin/…`.
- Building on `wc/v4`, `/wc-api/v3`, or the GraphQL Dual API — disabled or removed in 11.2.1. Use `wc/v3`.
- Leaving `$namespace` at the `WC_REST_Controller` default (`wc/v1`), or declaring the controller class at plugin load before WooCommerce's autoloader exists.
- Expecting WC API keys to work on `my-plugin/v1`. They only authenticate `wc/` and `wc-` namespaces; use Application Passwords or rename to `wc-my-plugin/v1`.
- Calling `wc_rest_check_manager_permissions( 'my_object' )` — only 7 keys are valid; anything else returns false with a warning.
- `'validate_callback' => 'is_numeric'` on a route arg: fatal on PHP 8. Use `'type' => 'integer', 'minimum' => 1`.
- Trusting route-arg validation inside `update_item()` when the route also serves `/batch` — batch sub-requests bypass it.
- A `register_rest_field()` `update_callback` that only calls `update_meta_data()` — the controller already saved; without `save()` the value is lost.
- `get_post_meta( $data['id'] )` in a REST `get_callback` — wrong storage with HPOS. Load the object with `wc_get_order()` / `wc_get_product()`.
- Expecting REST fields or `woocommerce_rest_prepare_*` data in the Checkout block. The block reads the Store API; use `woocommerce_store_api_register_endpoint_data()`.
- Putting secrets or extra PII in REST responses without remembering they are also webhook payloads.
- Registering a webhook topic like `order.published` (rejected since 10.9), or adding the resources/events filters after `init` (webhooks were already loaded, `set_topic()` stored `''`).
- Pointing a webhook at `localhost` in development — `wp_safe_remote_request()` refuses it.
- Naming an ability `woocommerce/…` (silently skipped) or calling `wp_register_ability()` outside `wp_abilities_api_init`.
- `'permission_callback' => '__return_true'` on an ability exposed to MCP.
- Documenting `/wp-json/woocommerce/mcp` + `X-MCP-API-Key` as the way to connect an agent — removed in 11.3.
