# Testing a WooCommerce extension (late 2026, WC 11.2.1)

What a WooCommerce extension can actually test with today, and how: PHPUnit on the WordPress test suite with WooCommerce loaded and **HPOS on**, Store API requests in-process, Action Scheduler assertions, wp-env 12, Playwright against the **Checkout block**, PHPStan with WooCommerce stubs, WooCommerce sniffs and Plugin Check. Everything assumes the skill's premises: blocks only, HPOS only, Action Scheduler only — no posts-storage or shortcode-checkout tests.

## What exists and what does not

- **`WC_Unit_Test_Case` and the `WC_Helper_*` classes are not shipped.** The wp.org zip of 11.2.1 has no `tests/` directory. They live only in the monorepo (`plugins/woocommerce/tests/legacy/framework/`), and their bootstrap depends on WooCommerce's own dev vendor and testing container. Packagist `woocommerce/woocommerce` is abandoned at 5.9.1.
- The official scaffold `@woocommerce/create-woo-extension` (2.1.0) generates only `phpunit/phpunit ^9.0` and a plain `PHPUnit\Framework\TestCase` sample.
- The devblog page "Testing extensions and maintaining quality code" (Puppeteer, `@woocommerce/e2e-environment`, PHPUnit 7.5) is stale. Don't follow it.
- So: bootstrap on WordPress's `WP_UnitTestCase`, load the installed WooCommerce, force HPOS, and write your own small helpers on public CRUD. WooCommerce's own suite runs with HPOS on by default; yours should too.

Versions verified for this file: PHPUnit 9.6 + `yoast/phpunit-polyfills` ^2.0 (the pair WooCommerce 11.2.1 itself uses; the WP 7.0 test bootstrap needs PHPUnit ≥ 5.7.21 and Polyfills ≥ 1.1.0 and is not documented for PHPUnit 10+), `@wordpress/env` 12.0.0 (Node ≥ 22.13), `@woocommerce/e2e-utils-playwright` 0.5.1 (peer `@playwright/test` ^1.63), `@wordpress/e2e-test-utils-playwright` 3.1.0, `php-stubs/woocommerce-stubs` v11.2.0 (there is no 11.2.1 tag), `szepeviktor/phpstan-wordpress` 2.0.4, `phpstan/phpstan` 2.x, `woocommerce/woocommerce-sniffs` 2.0.0 (requires WPCS ^3.4.1), Plugin Check 2.1.0.

## Layout and Composer

```
my-plugin/                    ← folder name = slug; wp-env mounts "." as wp-content/plugins/my-plugin
├── my-plugin.php
├── includes/
├── tests/
│   ├── bootstrap.php
│   ├── class-my-plugin-test-case.php
│   └── test-*.php
├── e2e/
├── composer.json
├── phpunit.xml.dist
├── phpstan.neon.dist
├── phpcs.xml.dist
├── .wp-env.json              ← dev + E2E environment (port 8888)
└── .wp-env.tests.json        ← disposable PHPUnit environment (port 8889)
```

```json
{
	"name": "my-vendor/my-plugin",
	"type": "wordpress-plugin",
	"require": {
		"php": ">=8.4"
	},
	"require-dev": {
		"phpunit/phpunit": "^9.6",
		"yoast/phpunit-polyfills": "^2.0",
		"phpstan/phpstan": "^2.1",
		"szepeviktor/phpstan-wordpress": "^2.0",
		"php-stubs/wordpress-stubs": "^7.1",
		"php-stubs/woocommerce-stubs": "^11.2",
		"woocommerce/woocommerce-sniffs": "^2.0"
	},
	"config": {
		"allow-plugins": {
			"dealerdirect/phpcodesniffer-composer-installer": true
		}
	},
	"scripts": {
		"test": "phpunit",
		"lint": "phpcs",
		"analyse": "phpstan analyse --memory-limit=1G"
	}
}
```

## wp-env 12: two environments, never one

wp-env generates `wp-tests-config.php` by **copying the site's `wp-config.php`**, so `WP_TESTS_DIR` (`/wordpress-phpunit` in the containers) points at the same database as the site. The WordPress test installer drops and recreates the core tables on every PHPUnit run. Run PHPUnit only in a dedicated environment started with `--config`; the old `testsEnvironment` option is deprecated (default `false`, WooCommerce's own `.wp-env.json` sets it to `false`), and without it there is no `tests-cli` container. `wp-env run` accepts `mysql`, `wordpress`, `cli`, `composer`, `phpmyadmin`.

`.wp-env.json` (development and E2E):

```json
{
	"$schema": "https://schemas.wp.org/trunk/wp-env.json",
	"core": null,
	"phpVersion": "8.4",
	"port": 8888,
	"plugins": [ "https://downloads.wordpress.org/plugin/woocommerce.11.2.1.zip", "." ],
	"config": {
		"WP_DEBUG_LOG": true
	},
	"lifecycleScripts": {
		"afterStart": "wp-env run cli wp option update woocommerce_coming_soon no && wp-env run cli wp wc hpos enable"
	}
}
```

`.wp-env.tests.json` (PHPUnit only — treat its database as disposable):

```json
{
	"$schema": "https://schemas.wp.org/trunk/wp-env.json",
	"core": null,
	"phpVersion": "8.4",
	"port": 8889,
	"plugins": [ "https://downloads.wordpress.org/plugin/woocommerce.11.2.1.zip", "." ]
}
```

```bash
npx wp-env start                                   # dev/E2E site on :8888
npx wp-env start --config=.wp-env.tests.json       # PHPUnit site on :8889
npx wp-env run cli --config=.wp-env.tests.json --env-cwd=wp-content/plugins/my-plugin composer install
npx wp-env run cli --config=.wp-env.tests.json --env-cwd=wp-content/plugins/my-plugin vendor/bin/phpunit
npx wp-env run cli --config=.wp-env.tests.json --env-cwd=wp-content/plugins/my-plugin vendor/bin/phpunit -- --filter Store_Api
```

Notes: new shops get `woocommerce_coming_soon = yes`, which hides the storefront from logged-out E2E visitors — the `afterStart` turns it off. `wp wc hpos enable` is idempotent (warns "already enabled"); `--for-new-shop` errors on every later start, and the `wc cot *` aliases are deprecated since 8.9. wp-env defines `WP_ENVIRONMENT_TYPE=local`, which makes Application Passwords available over plain HTTP. Pin the WooCommerce zip to the version you test against; add a second config (or CI matrix entry) for the next release.

## PHPUnit configuration and HPOS-on bootstrap

```xml
<?xml version="1.0"?>
<phpunit
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.6/phpunit.xsd"
	bootstrap="tests/bootstrap.php"
	colors="true"
	convertDeprecationsToExceptions="true"
>
	<testsuites>
		<testsuite name="my-plugin">
			<directory prefix="test-" suffix=".php">./tests/</directory>
		</testsuite>
	</testsuites>
</phpunit>
```

```php
<?php
/**
 * PHPUnit bootstrap: WordPress test suite + WooCommerce with HPOS authoritative.
 *
 * @package My_Plugin
 */

$my_plugin_tests_dir = getenv( 'WP_TESTS_DIR' );
if ( ! $my_plugin_tests_dir ) {
	$my_plugin_tests_dir = rtrim( sys_get_temp_dir(), '/\\' ) . '/wordpress-tests-lib';
}

define( 'WP_TESTS_PHPUNIT_POLYFILLS_PATH', dirname( __DIR__ ) . '/vendor/yoast/phpunit-polyfills' );

require_once $my_plugin_tests_dir . '/includes/functions.php';

tests_add_filter(
	'muplugins_loaded',
	static function () {
		require WP_PLUGIN_DIR . '/woocommerce/woocommerce.php';
		require dirname( __DIR__ ) . '/my-plugin.php';
	}
);

tests_add_filter(
	'setup_theme',
	static function () {
		// The test installer just recreated the WP tables: install WooCommerce into them.
		WC_Install::install();

		// HPOS authoritative, sync off. Updating this option creates the wc_orders* tables
		// (CustomOrdersTableController::process_pre_update_option, hooked when WooCommerce loads).
		// Do it here: once WP_UnitTestCase runs, CREATE TABLE is turned into temporary tables.
		update_option( 'woocommerce_custom_orders_table_data_sync_enabled', 'no' );
		update_option( 'woocommerce_custom_orders_table_enabled', 'yes' );

		// Reload roles so shop_manager/customer caps added by the installer exist.
		$GLOBALS['wp_roles'] = null; // phpcs:ignore WordPress.WP.GlobalVariablesOverride.Prohibited
		wp_roles();
	}
);

// WC sessions call setcookie(); in CLI that would raise "headers already sent" notices.
tests_add_filter( 'woocommerce_set_cookie_enabled', '__return_false' );

require $my_plugin_tests_dir . '/includes/bootstrap.php';
```

Why not rely on auto-enable: `WC_Install::maybe_enable_hpos()` runs only for a newly installed shop, after `woocommerce_init`, and only when no active WooCommerce-aware plugin is incompatible **or uncertain**. Set it explicitly and assert it (below). The compatibility test needs the plugin to live in `WP_PLUGIN_DIR` (true on wp-env), because WooCommerce resolves `__FILE__` against `get_plugins()`.

## Base test case and your own helpers

```php
<?php
/**
 * Base class: fresh REST server, a WC session and cart, CRUD factories.
 *
 * @package My_Plugin
 */

abstract class My_Plugin_Test_Case extends WP_UnitTestCase {

	public function set_up() {
		parent::set_up();
		$GLOBALS['wp_rest_server'] = null; // phpcs:ignore WordPress.WP.GlobalVariablesOverride.Prohibited -- Rebuild routes per test.
		WC()->initialize_session();       // Cookies are disabled in the bootstrap; the session lives in memory.
		WC()->initialize_cart();
	}

	public function tear_down() {
		if ( WC()->cart ) {
			WC()->cart->empty_cart(); // Fine in tests; never in a gateway.
		}
		$GLOBALS['wp_rest_server'] = null; // phpcs:ignore WordPress.WP.GlobalVariablesOverride.Prohibited
		parent::tear_down();
	}

	protected function create_simple_product( array $props = array() ): WC_Product_Simple {
		$product = new WC_Product_Simple();
		$product->set_props(
			array_merge(
				array(
					'name'          => 'Test wine',
					'regular_price' => '9990',
					'status'        => 'publish',
				),
				$props
			)
		);
		$product->save();
		return $product;
	}

	protected function create_order( WC_Product $product, int $qty = 1 ): WC_Order {
		$order = wc_create_order( array( 'status' => 'pending' ) );
		$order->add_product( $product, $qty );
		$order->set_billing_email( 'buyer@example.com' );
		$order->set_payment_method( 'bacs' );
		$order->calculate_totals();
		$order->save();
		return $order;
	}

	protected function store_api( string $method, string $route, array $params = array(), bool $with_nonce = true ): WP_REST_Response {
		$request = new WP_REST_Request( $method, '/wc/store/v1' . $route );
		if ( $with_nonce && 'GET' !== $method ) {
			$request->set_header( 'Nonce', wp_create_nonce( 'wc_store_api' ) ); // What the browser reads from the Nonce response header.
		}
		foreach ( $params as $key => $value ) {
			$request->set_param( $key, $value );
		}
		return rest_do_request( $request );
	}
}
```

`WP_UnitTestCase` fails a test on any unexpected `_doing_it_wrong()` or deprecation notice — including WooCommerce's (`wc_doing_it_wrong`). Treat those failures as real; use `setExpectedIncorrectUsage()` only for a call you test on purpose. Never call `woocommerce_store_api_disable_nonce_check` in tests you want to mean something: send the nonce, as the browser does.

## Code under test (excerpt of `my-plugin.php`)

```php
define( 'MY_PLUGIN_FILE', __FILE__ );

add_action(
	'before_woocommerce_init',
	static function () {
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', MY_PLUGIN_FILE, true );
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'cart_checkout_blocks', MY_PLUGIN_FILE, true );
	}
);

// Paid orders sync to the ERP in the background, once.
add_action( 'woocommerce_payment_complete', 'my_plugin_queue_order_sync' );
add_action( 'my_plugin_sync_order', 'my_plugin_sync_order' );

function my_plugin_queue_order_sync( $order_id ) {
	as_enqueue_async_action( 'my_plugin_sync_order', array( (int) $order_id ), 'my-plugin', true );
}

function my_plugin_sync_order( $order_id ) {
	$order = wc_get_order( $order_id );
	if ( ! $order ) {
		return;
	}
	$response = wp_remote_post( 'https://erp.example.com/orders', array( 'body' => wp_json_encode( array( 'id' => $order->get_id() ) ) ) );
	if ( 200 !== wp_remote_retrieve_response_code( $response ) ) {
		throw new Exception( 'ERP sync failed' ); // Action Scheduler marks the action failed; reschedule with backoff yourself.
	}
	$order->update_meta_data( '_my_plugin_synced_at', gmdate( 'c' ) );
	$order->save();
}

// Members-only products cannot be added by guests (Store API add-to-cart).
add_action( 'woocommerce_store_api_validate_add_to_cart', 'my_plugin_validate_add_to_cart', 10, 2 );

function my_plugin_validate_add_to_cart( $product, $request ) {
	if ( 'yes' === $product->get_meta( '_my_plugin_members_only' ) && ! is_user_logged_in() ) {
		throw new \Automattic\WooCommerce\StoreApi\Exceptions\RouteException( 'my_plugin_members_only', __( 'Log in to buy this wine.', 'my-plugin' ), 403 );
	}
}

// Custom shopper endpoint in the Store API namespace: wc/store/v1/my-plugin/gift-note.
add_action( 'rest_api_init', 'my_plugin_register_store_routes' );

function my_plugin_register_store_routes() {
	register_rest_route(
		'wc/store/v1',
		'/my-plugin/gift-note',
		array(
			array(
				'methods'             => WP_REST_Server::READABLE,
				'callback'            => 'my_plugin_store_get_gift_note',
				'permission_callback' => '__return_true', // Scoped to the shopper's own session.
			),
			array(
				'methods'             => WP_REST_Server::CREATABLE,
				'callback'            => 'my_plugin_store_set_gift_note',
				'permission_callback' => 'my_plugin_store_check_nonce',
				'args'                => array(
					'note' => array(
						'type'              => 'string',
						'maxLength'         => 200,
						'required'          => true,
						'sanitize_callback' => 'sanitize_textarea_field',
					),
				),
			),
		)
	);
}

// Core checks the nonce only in its own cart routes; custom write routes do it themselves.
function my_plugin_store_check_nonce( WP_REST_Request $request ) {
	$nonce = $request->get_header( 'Nonce' );
	if ( ! $nonce || ! wp_verify_nonce( $nonce, 'wc_store_api' ) ) {
		return new WP_Error( 'my_plugin_invalid_nonce', __( 'Missing or invalid Nonce header.', 'my-plugin' ), array( 'status' => 401 ) );
	}
	return true;
}

function my_plugin_store_get_gift_note() {
	wc_load_cart(); // Custom routes do not get the session/cart loaded for them.
	$response = rest_ensure_response( array( 'note' => (string) WC()->session->get( 'my_plugin_gift_note', '' ) ) );
	$response->header( 'Cache-Control', 'no-store' );
	return $response;
}

function my_plugin_store_set_gift_note( WP_REST_Request $request ) {
	wc_load_cart();
	WC()->session->set( 'my_plugin_gift_note', $request['note'] );
	return my_plugin_store_get_gift_note();
}
```

## Tests that mean something

```php
<?php
/**
 * Compatibility, HPOS storage and order lifecycle.
 *
 * @package My_Plugin
 */

use Automattic\WooCommerce\Utilities\FeaturesUtil;
use Automattic\WooCommerce\Utilities\OrderUtil;

class Test_My_Plugin_Orders extends My_Plugin_Test_Case {

	public function test_declares_hpos_and_block_checkout_compatibility() {
		$features = FeaturesUtil::get_compatible_features_for_plugin( plugin_basename( MY_PLUGIN_FILE ) ); // Valid after woocommerce_init.

		$this->assertContains( 'custom_order_tables', $features['compatible'] );
		$this->assertContains( 'cart_checkout_blocks', $features['compatible'] );
	}

	public function test_suite_runs_on_hpos() {
		$this->assertTrue( OrderUtil::custom_orders_table_usage_is_enabled() );
	}

	public function test_order_meta_is_stored_in_hpos_tables_only() {
		$order = $this->create_order( $this->create_simple_product() );
		$order->update_meta_data( '_my_plugin_flag', 'yes' );
		$order->save();

		$this->assertSame( 'yes', wc_get_order( $order->get_id() )->get_meta( '_my_plugin_flag' ) );
		$this->assertSame( '', get_post_meta( $order->get_id(), '_my_plugin_flag', true ), 'Sync is off: nothing may read orders from wp_postmeta.' );
	}

	public function test_payment_complete_moves_physical_order_to_processing() {
		$order = $this->create_order( $this->create_simple_product() );
		$order->payment_complete( 'txn_123' );

		// needs_processing() is true unless every item is virtual AND downloadable.
		$this->assertSame( 'processing', wc_get_order( $order->get_id() )->get_status() );
	}

	public function test_payment_complete_moves_virtual_downloadable_order_to_completed() {
		$product = $this->create_simple_product(
			array(
				'virtual'      => true,
				'downloadable' => true,
			)
		);
		$order   = $this->create_order( $product );
		$order->payment_complete( 'txn_456' );

		$this->assertSame( 'completed', wc_get_order( $order->get_id() )->get_status() );
	}

	public function test_payment_complete_enqueues_exactly_one_sync_job() {
		$order = $this->create_order( $this->create_simple_product() );
		$order->payment_complete( 'txn_789' );
		my_plugin_queue_order_sync( $order->get_id() ); // Second attempt: unique = hook + group + args.

		$ids = as_get_scheduled_actions(
			array(
				'hook'   => 'my_plugin_sync_order',
				'args'   => array( $order->get_id() ),
				'group'  => 'my-plugin',
				'status' => ActionScheduler_Store::STATUS_PENDING,
			),
			'ids'
		);
		$this->assertCount( 1, $ids );
		$this->assertNotFalse( as_next_scheduled_action( 'my_plugin_sync_order', array( $order->get_id() ), 'my-plugin' ) );
	}

	public function test_sync_job_marks_order_when_erp_answers_200() {
		$order = $this->create_order( $this->create_simple_product() );
		add_filter(
			'pre_http_request',
			static function () {
				return array(
					'headers'  => array(),
					'body'     => '{}',
					'response' => array(
						'code'    => 200,
						'message' => 'OK',
					),
					'cookies'  => array(),
				);
			}
		);

		do_action( 'my_plugin_sync_order', $order->get_id() ); // Run the job body directly; AS's runner is not under test.

		$this->assertNotEmpty( wc_get_order( $order->get_id() )->get_meta( '_my_plugin_synced_at' ) );
	}
}
```

```php
<?php
/**
 * Store API: core routes and the custom wc/store/v1/my-plugin route.
 *
 * @package My_Plugin
 */

class Test_My_Plugin_Store_Api extends My_Plugin_Test_Case {

	public function test_product_prices_are_in_minor_units() {
		$product  = $this->create_simple_product();
		$response = $this->store_api( 'GET', '/products/' . $product->get_id() );
		$prices   = $response->get_data()['prices'];

		$this->assertSame( 200, $response->get_status() );
		$this->assertSame( 9990 * ( 10 ** $prices['currency_minor_unit'] ), (int) $prices['price'] );
	}

	public function test_add_item_requires_nonce() {
		$product  = $this->create_simple_product();
		$response = $this->store_api( 'POST', '/cart/add-item', array( 'id' => $product->get_id(), 'quantity' => 1 ), false );

		$this->assertSame( 401, $response->get_status() );
		$this->assertSame( 'woocommerce_rest_missing_nonce', $response->get_data()['code'] );
	}

	public function test_guest_cannot_add_members_only_product() {
		$product = $this->create_simple_product();
		$product->update_meta_data( '_my_plugin_members_only', 'yes' );
		$product->save();

		$response = $this->store_api( 'POST', '/cart/add-item', array( 'id' => $product->get_id(), 'quantity' => 1 ) );

		$this->assertSame( 403, $response->get_status() );
		$this->assertSame( 'my_plugin_members_only', $response->get_data()['code'] );
	}

	public function test_guest_can_add_regular_product() {
		$product  = $this->create_simple_product();
		$response = $this->store_api( 'POST', '/cart/add-item', array( 'id' => $product->get_id(), 'quantity' => 2 ) );

		$this->assertSame( 201, $response->get_status() );
		$this->assertSame( 2, $response->get_data()['items_count'] );
		$this->assertSame( 'no-store', $response->get_headers()['Cache-Control'] );
	}

	public function test_custom_route_rejects_writes_without_nonce() {
		$response = $this->store_api( 'POST', '/my-plugin/gift-note', array( 'note' => 'Happy birthday' ), false );

		$this->assertSame( 401, $response->get_status() );
	}

	public function test_custom_route_stores_note_in_shopper_session() {
		$this->store_api( 'POST', '/my-plugin/gift-note', array( 'note' => '<b>Happy</b> birthday' ) );
		$response = $this->store_api( 'GET', '/my-plugin/gift-note' );

		$this->assertSame( 200, $response->get_status() );
		$this->assertSame( 'Happy birthday', $response->get_data()['note'] );
		$this->assertSame( 'no-store', $response->get_headers()['Cache-Control'] );
	}

	public function test_custom_route_validates_length() {
		$response = $this->store_api( 'POST', '/my-plugin/gift-note', array( 'note' => str_repeat( 'x', 201 ) ) );

		$this->assertSame( 400, $response->get_status() );
	}
}
```

In-process requests skip what depends on the real URL: `WC()->is_store_api_request()` reads `$_SERVER['REQUEST_URI']`, so CORS headers, the Cart-Token session swap and rate limiting are not exercised here — cover them in E2E. Unit-test checkout validation callbacks (`validate_callback` of additional fields, `woocommerce_checkout_validate_order_before_payment`) directly, and the whole place-order flow in E2E. For `wc/v3` controllers use the same `rest_do_request()` pattern with `wp_set_current_user( self::factory()->user->create( array( 'role' => 'shop_manager' ) ) )`; for abilities, `wp_get_ability( 'my-plugin/x' )->execute( $input )` returns `WP_Error( 'ability_invalid_permissions' )` for a user without access.

## E2E: Playwright against the Checkout block

New installs create Cart and Checkout pages **with block content**. `@woocommerce/e2e-utils-playwright` is 0.x ("can introduce breaking changes") — pin it. Its checkout helpers only target the block checkout: `fillBillingCheckoutBlocks` / `fillShippingCheckoutBlocks` (labels "Billing address" / "Shipping address"; fields `firstName`, `lastName`, `address`, `city`, `state`, `zip`, `country` (option value), `phone`, `isPostalCode`… — **no email**, fill "Email address" yourself). Others: `addAProductToCart( page, productId, qty )` (visits `shop/?add-to-cart=` and waits for `/wc/store/v1/cart`), `getOrderIdFromUrl( page )`, `createClient( baseURL, auth )` with `WC_API_PATH = 'wc/v3'`.

```bash
npm i -D @playwright/test@^1.63 @woocommerce/e2e-utils-playwright@0.5.1 @wordpress/e2e-test-utils-playwright@3.1.0
npx playwright install chromium
```

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig( {
	testDir: './e2e',
	globalSetup: './e2e/global-setup.ts',
	use: { baseURL: 'http://localhost:8888/' },
	retries: process.env.CI ? 2 : 0,
} );
```

```ts
// e2e/global-setup.ts — Application Password (works over HTTP because wp-env is "local"), COD on.
import { execSync } from 'node:child_process';
import { createClient, WC_API_PATH } from '@woocommerce/e2e-utils-playwright';

export default async function globalSetup() {
	const out = execSync( 'npx wp-env run cli wp user application-password create admin e2e --porcelain' ).toString().trim();
	process.env.WP_APP_PASSWORD = out.split( '\n' ).pop();

	const api = createClient( 'http://localhost:8888/', { type: 'basic', username: 'admin', password: process.env.WP_APP_PASSWORD! } );
	await api.put( `${ WC_API_PATH }/payment_gateways/cod`, { enabled: true } );
}
```

```ts
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
import { createClient, WC_API_PATH, addAProductToCart, fillBillingCheckoutBlocks, getOrderIdFromUrl } from '@woocommerce/e2e-utils-playwright';

test( 'guest checkout (Checkout block) creates an HPOS order with plugin data', async ( { page, baseURL } ) => {
	const api = createClient( baseURL!, { type: 'basic', username: 'admin', password: process.env.WP_APP_PASSWORD! } );
	// Virtual product: no shipping zone needed, and the block shows the billing address form.
	const { data: product } = await api.post( `${ WC_API_PATH }/products`, { name: 'E2E Wine', regular_price: '9990', virtual: true } );

	await addAProductToCart( page, String( product.id ) );
	await page.goto( 'checkout/' );
	await page.getByLabel( 'Email address' ).fill( 'e2e@example.com' );
	await fillBillingCheckoutBlocks( page, {
		firstName: 'Ana', lastName: 'Rojas', address: '1 Main St', city: 'San Francisco',
		country: 'US', state: 'CA', zip: '94107',
	} );
	await page.getByRole( 'button', { name: 'Place order' } ).click();

	await expect( page ).toHaveURL( /order-received/ );
	const orderId = getOrderIdFromUrl( page );
	const { data: order } = await api.get( `${ WC_API_PATH }/orders/${ orderId }` );
	expect( order.status ).toBe( 'processing' ); // COD → processing.
} );
```

For physical products add a shipping method first (e.g. `POST wc/v3/shipping/zones/0/methods` with `{ method_id: 'free_shipping' }`) and use `fillShippingCheckoutBlocks`. Labels are the English UI strings of 11.2; a site in another language needs its own locators. Assert results through `wc/v3` or WP-CLI, never by querying `wp_posts`.

## Static analysis, standards, Plugin Check

`phpstan.neon.dist`:

```neon
includes:
	- vendor/szepeviktor/phpstan-wordpress/extension.neon
parameters:
	level: 6
	paths:
		- my-plugin.php
		- includes/
	scanFiles:
		- vendor/php-stubs/woocommerce-stubs/woocommerce-stubs.php
		- vendor/php-stubs/woocommerce-stubs/woocommerce-packages-stubs.php
```

`phpcs.xml.dist` (ruleset `WooCommerce-Core` = WooCommerce sniffs + PHPCompatibilityWP + WordPress):

```xml
<?xml version="1.0"?>
<ruleset name="My Plugin">
	<file>.</file>
	<exclude-pattern>/vendor/*</exclude-pattern>
	<exclude-pattern>/node_modules/*</exclude-pattern>
	<exclude-pattern>/build/*</exclude-pattern>
	<arg name="extensions" value="php"/>
	<arg value="sp"/>
	<config name="testVersion" value="8.4-"/>
	<config name="minimum_wp_version" value="7.0"/>
	<rule ref="WooCommerce-Core"/>
	<rule ref="WordPress.WP.I18n">
		<properties>
			<property name="text_domain" type="array">
				<element value="my-plugin"/>
			</property>
		</properties>
	</rule>
	<rule ref="WordPress.NamingConventions.PrefixAllGlobals">
		<properties>
			<property name="prefixes" type="array">
				<element value="my_plugin"/>
				<element value="My_Plugin"/>
			</property>
		</properties>
	</rule>
</ruleset>
```

This skill targets PHP 8.4+, so set `testVersion` to `8.4-`. Plugin Check (wp.org `plugin-check` 2.1.0):

```bash
npx wp-env run cli wp plugin install plugin-check --activate
npx wp-env run cli wp plugin check my-plugin
```

## Required plugin headers

```php
/**
 * Plugin Name:          My Plugin for WooCommerce
 * Requires at least:    7.0
 * Requires PHP:         8.4
 * Requires Plugins:     woocommerce
 * WC requires at least: 11.2.1
 * WC tested up to:      11.2
 * Text Domain:          my-plugin
 */
```

- WooCommerce reads `WC requires at least` and `WC tested up to`. A plugin is **WooCommerce-aware iff `WC tested up to` is non-empty**; aware plugins that do not declare `custom_order_tables` count as *uncertain* and block HPOS auto-enable on new shops. Always pair the header with the compatibility declarations shown above.
- `Requires Plugins: woocommerce` is the WP core dependency header (slug = folder name). Raise `WC requires at least` to the version of the newest API you call (10.9 for `AbilityDefinition`, 11.2 for `woocommerce_store_api_cart_item_quantity_validation`).
- WooCommerce's own header doc still shows `8.0 / 8.3` and omits `Requires Plugins` — stale.

## Common mistakes

- `extends WC_Unit_Test_Case` or `WC_Helper_Order::create_order()` — not shipped; the class never loads.
- Running PHPUnit in the same wp-env as your dev site: the test installer drops its core tables.
- `wp-env run tests-cli …` or `"testsEnvironment": true` — deprecated path; use a second config with `--config`.
- No HPOS in the suite, or relying on auto-enable. Set `woocommerce_custom_orders_table_enabled` in the bootstrap and assert `OrderUtil::custom_orders_table_usage_is_enabled()`.
- Asserting `completed` after `payment_complete()` for a physical product. It is `processing`; only virtual **and** downloadable items skip processing.
- "Testing" HPOS compatibility with `class_exists( FeaturesUtil::class )`. Assert `get_compatible_features_for_plugin()` contains `custom_order_tables` (after `woocommerce_init`).
- Testing checkout through `woocommerce_checkout_process` and `$_POST`, or reading `wc_get_notices()` (returns `[]` without a session). The Checkout block posts to `/wc/store/v1/checkout`: unit-test your callbacks, E2E the flow.
- Disabling the Store API nonce check in tests; or forgetting that custom `wc/store/v1/my-plugin` write routes must check `Nonce` themselves.
- `get_post_meta()` / `WP_Query` on orders in assertions.
- Asserting Action Scheduler effects by running WP-Cron. Assert with `as_get_scheduled_actions()` / `as_next_scheduled_action()` and call the job callback directly.
- E2E with coming-soon mode on (logged-out visitors see the coming-soon page), against a shortcode checkout, or with an unpinned `@woocommerce/e2e-utils-playwright` 0.x.
- Missing `WC tested up to` + compatibility declarations: the plugin is flagged and blocks HPOS on new shops.
