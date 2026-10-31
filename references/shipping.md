# Shipping methods (Checkout block)

How to write a shipping method for WooCommerce 11.2.1 that works in shipping zones and in the Cart/Checkout blocks: the `WC_Shipping_Method` contract, a complete instance-based method with a remote quote, the rate pipeline, local pickup in blocks, and how rates reach the browser through the Store API. Rates are never rendered into cached HTML: the blocks fetch them from `/wc/store/v1/cart`.

## Contract

`includes/abstracts/abstract-wc-shipping-method.php`, extends `WC_Settings_API`.

| Member | Notes |
|---|---|
| `$id` | Unique method id (rate ids default to `{id}:{instance_id}`). |
| `$instance_id` | Set from the constructor argument; one instance per zone method. |
| `$method_title`, `$method_description` | Shown in the zone's "Add shipping method" UI. |
| `$title` | Shopper-facing label. Read it from an **instance** `title` field so each zone can rename it. |
| `$supports` | Default `array( 'settings' )`. Core features: `shipping-zones` (instances), `instance-settings`, `instance-settings-modal` (settings open in a modal in the zones UI), `settings` (global screen), plus `local-pickup` (see below). |
| `$instance_form_fields` | Per-zone fields. Filter `woocommerce_shipping_instance_form_fields_{id}`. Must be set **before** you call `get_option()`. |
| `$tax_status` | `ProductTaxStatus::TAXABLE` (default) or `NONE`; `is_taxable()` also needs taxes enabled. |
| `$rates` | Public array filled by `add_rate()`, keyed by rate id. |
| `calculate_shipping( $package )` | Call `add_rate()` zero or more times. Zero rates = method not offered. |
| `is_available( $package )` | `is_enabled()` (+ legacy country rules for non-zone methods); filter `woocommerce_shipping_{id}_is_available`. |
| `get_option( $key )` | Instance setting first (when the key is an instance field), else global setting (filter `woocommerce_shipping_{id}_option`). |
| `process_admin_options()` | Saves the form. Hook it on `woocommerce_update_options_shipping_{id}` as core does. |

Registration: `woocommerce_shipping_methods` receives `array( method_id => class_name )`. Load the class file on `woocommerce_shipping_init` (fired by `WC_Shipping` before it reads the filter).

## A complete method with a remote quote

Weight-based courier: asks the carrier API for a price, caches it, falls back to a local formula when the API is down, and attaches visible and hidden metadata plus a delivery estimate.

```php
<?php
/**
 * Plugin Name:          My Plugin Courier
 * Requires at least:    7.0
 * Requires PHP:         8.4
 * Requires Plugins:     woocommerce
 * Text Domain:          my-plugin
 */

defined( 'ABSPATH' ) || exit;

add_action(
	'before_woocommerce_init',
	static function () {
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
		\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'cart_checkout_blocks', __FILE__, true );
	}
);

add_action(
	'woocommerce_shipping_init',
	static function () {
		require_once __DIR__ . '/includes/class-my-plugin-courier-method.php';
	}
);

add_filter(
	'woocommerce_shipping_methods',
	static function ( array $methods ) {
		$methods['my_plugin_courier'] = 'My_Plugin_Courier_Method';
		return $methods;
	}
);
```

`includes/class-my-plugin-courier-method.php`:

```php
<?php
defined( 'ABSPATH' ) || exit;

use Automattic\WooCommerce\Enums\ProductTaxStatus;

class My_Plugin_Courier_Method extends WC_Shipping_Method {

	public function __construct( $instance_id = 0 ) {
		parent::__construct( $instance_id );
		$this->id                 = 'my_plugin_courier';
		$this->method_title       = __( 'My courier', 'my-plugin' );
		$this->method_description = __( 'Live rates from the courier API, with a weight-based fallback.', 'my-plugin' );
		$this->supports           = array( 'shipping-zones', 'instance-settings', 'instance-settings-modal' );

		$this->instance_form_fields = array(
			'title'      => array(
				'title'   => __( 'Name', 'my-plugin' ),
				'type'    => 'text',
				'default' => __( 'Home delivery', 'my-plugin' ),
			),
			'tax_status' => array(
				'title'   => __( 'Tax status', 'my-plugin' ),
				'type'    => 'select',
				'default' => ProductTaxStatus::TAXABLE,
				'options' => array(
					ProductTaxStatus::TAXABLE => __( 'Taxable', 'my-plugin' ),
					ProductTaxStatus::NONE    => __( 'None', 'my-plugin' ),
				),
			),
			'service'    => array(
				'title'   => __( 'Courier service code', 'my-plugin' ),
				'type'    => 'text',
				'default' => 'standard',
			),
			'base_cost'  => array(
				'title'   => __( 'Fallback base cost', 'my-plugin' ),
				'type'    => 'price',
				'default' => '3990',
			),
			'per_kg'     => array(
				'title'   => __( 'Fallback cost per kg', 'my-plugin' ),
				'type'    => 'price',
				'default' => '990',
			),
			'days'       => array(
				'title'   => __( 'Fallback delivery estimate', 'my-plugin' ),
				'type'    => 'text',
				'default' => __( '2 to 4 business days', 'my-plugin' ),
			),
		);

		$this->title      = $this->get_option( 'title' );
		$this->tax_status = $this->get_option( 'tax_status' );

		add_action( 'woocommerce_update_options_shipping_' . $this->id, array( $this, 'process_admin_options' ) );
	}

	public function calculate_shipping( $package = array() ) {
		$kg = 0.0;
		foreach ( $package['contents'] as $item ) {
			$kg += (float) $item['data']->get_weight() * (int) $item['quantity'];
		}
		$kg = (float) wc_get_weight( $kg, 'kg' ); // Store weight unit -> kg.

		$quote = $this->get_remote_quote( $package['destination'], $kg );
		if ( null === $quote ) {
			$quote = array(
				'cost'     => (float) $this->get_option( 'base_cost' ) + ceil( $kg ) * (float) $this->get_option( 'per_kg' ),
				'eta'      => (string) $this->get_option( 'days' ),
				'quote_id' => '',
			);
		}

		$rate_id = $this->get_rate_id(); // my_plugin_courier:{instance_id}
		$this->add_rate(
			array(
				'id'             => $rate_id,
				'label'          => $this->title,
				'cost'           => $quote['cost'],
				'package'        => $package, // Adds a visible "Items" meta line.
				'price_decimals' => wc_get_price_decimals(), // Round to the store's decimals (0 for CLP).
				'meta_data'      => array(
					__( 'Weight', 'my-plugin' ) => wc_format_decimal( $kg, 2 ) . ' kg', // Visible: order, emails, Store API.
					'_my_plugin_quote_id'       => $quote['quote_id'],                 // Hidden: "_" prefix.
					'_my_plugin_service'        => $this->get_option( 'service' ),
				),
			)
		);

		// Description and delivery time are WC_Shipping_Rate props (9.2), not add_rate() args.
		if ( isset( $this->rates[ $rate_id ] ) ) {
			$this->rates[ $rate_id ]->set_delivery_time( $quote['eta'] );
			$this->rates[ $rate_id ]->set_description( __( 'Tracked delivery to your door.', 'my-plugin' ) );
		}
	}

	/**
	 * Short timeout, transient cache, negative cache. Returns null on any failure.
	 *
	 * @return array{cost: float, eta: string, quote_id: string}|null
	 */
	private function get_remote_quote( array $destination, float $kg ): ?array {
		$key    = 'my_plugin_quote_' . md5(
			wp_json_encode(
				array(
					$this->instance_id,
					$this->get_option( 'service' ),
					$destination['country'] ?? '',
					$destination['state'] ?? '',
					$destination['postcode'] ?? '',
					$destination['city'] ?? '',
					round( $kg, 1 ),
				)
			)
		);
		$cached = get_transient( $key );
		if ( is_array( $cached ) ) {
			return empty( $cached['failed'] ) ? $cached : null;
		}

		$response = wp_remote_post(
			'https://api.example-courier.test/v1/quotes',
			array(
				'timeout' => 3, // This runs inside a shopper's Store API request.
				'headers' => array( 'Content-Type' => 'application/json' ),
				'body'    => wp_json_encode(
					array(
						'service'     => $this->get_option( 'service' ),
						'destination' => $destination,
						'weight_kg'   => round( $kg, 3 ),
					)
				),
			)
		);
		$data = is_wp_error( $response ) ? null : json_decode( wp_remote_retrieve_body( $response ), true );

		if ( 200 !== (int) wp_remote_retrieve_response_code( $response ) || ! isset( $data['price'] ) ) {
			wc_get_logger()->warning( 'Courier quote failed', array( 'source' => 'my-plugin' ) );
			set_transient( $key, array( 'failed' => true ), 5 * MINUTE_IN_SECONDS ); // Do not hammer a dead API.
			return null;
		}

		$quote = array(
			'cost'     => (float) $data['price'],
			'eta'      => sanitize_text_field( (string) ( $data['eta'] ?? '' ) ),
			'quote_id' => sanitize_text_field( (string) ( $data['id'] ?? '' ) ),
		);
		set_transient( $key, $quote, 30 * MINUTE_IN_SECONDS );
		return $quote;
	}
}
```

Notes:

- Keep the instance `title` field: without it `get_option( 'title' )` falls back to the global setting and merchants cannot rename the method per zone.
- `type => 'price'` fields use the store's decimal separator and are validated by `WC_Settings_API::validate_price_field()`.
- Never call a remote API without a timeout inside `calculate_shipping()`: it blocks `/cart/update-customer`, `/cart/select-shipping-rate` and checkout.
- Do not call a remote API from Action Scheduler to "pre-warm" rates per shopper; the transient keyed by destination + weight is enough.

## `add_rate()` arguments

| Arg | Default | Notes |
|---|---|---|
| `id` | `get_rate_id()` | Pass a suffix-based id (`get_rate_id( 'express' )`) when one instance adds several rates. |
| `label` | `''` | Required; empty id or label = no rate. |
| `cost` | `'0'` | Number, or an array of per-item costs (summed). |
| `taxes` | `''` | `''` = calculated for you (if taxable), an array = your own taxes, `false` = none. |
| `calc_tax` | `'per_order'` | `'per_item'` needs `cost` as an array. |
| `meta_data` | `array()` | Key/value pairs copied to the order's shipping line item. Keys starting with `_` are hidden from the Store API and from the order/email item display. |
| `package` | `false` | Pass `$package` to add an "Items" meta line listing the products. |
| `price_decimals` | `false` | `false` = no rounding. Pass `wc_get_price_decimals()` to round. |

Filters: `woocommerce_shipping_method_add_rate_args` (args before the rate is built), `woocommerce_shipping_method_add_rate` (the `WC_Shipping_Rate`, another place to call `set_description()`/`set_delivery_time()`), `woocommerce_shipping_prices_include_tax` (10.6: treat entered costs as gross and convert to net).

## Rate pipeline and filters

```
WC_Shipping::calculate_shipping_for_package( $package )
  session "shipping_for_package_{n}" has the same package hash?  ── yes ──► reuse stored rates
  (option woocommerce_shipping_debug_mode = yes always recalculates)
  no ▼
  for each method in the zone (non-shippable destination: pickup methods only)
     woocommerce_before_get_rates_for_package( $package, $method )
     $method->get_rates_for_package()  ─► is_available() ─► calculate_shipping()
     woocommerce_after_get_rates_for_package( $package, $method )
  option woocommerce_shipping_hide_rates_when_free = yes and a free_shipping rate exists
     ─► keep free_shipping + local pickup rates only
  woocommerce_package_rates( $rates, $package )
  store in session with the package hash
```

- Packages come from `woocommerce_cart_shipping_packages` and can be split; `woocommerce_shipping_packages` filters the final list. The Store API and the blocks handle several packages.
- Because rates are cached per package hash in the session, a settings change does not show until the cart, destination or method set changes. Use debug mode while developing.
- "Hide other rates when free shipping is available" is a core setting (Settings › Shipping › Shipping settings). Do not reimplement it.

Filtering rates, e.g. remove express for a shipping class:

```php
add_filter(
	'woocommerce_package_rates',
	static function ( array $rates, array $package ) {
		foreach ( $package['contents'] as $item ) {
			if ( 'fragile' === $item['data']->get_shipping_class() ) {
				foreach ( $rates as $rate_id => $rate ) {
					if ( 'my_plugin_courier' === $rate->get_method_id() && false !== strpos( $rate_id, ':express' ) ) {
						unset( $rates[ $rate_id ] );
					}
				}
				break;
			}
		}
		return $rates;
	},
	10,
	2
);
```

Returning a non-array resets to an empty array (no rates). Never read `WC()->cart` in this filter for anything you could take from `$package`.

## Local pickup in blocks

**Core block local pickup** (method id `pickup_location`) is configured under Settings › Shipping › Local pickup: settings in `woocommerce_pickup_location_settings`, locations in `pickup_location_pickup_locations`. It is registered only when the Checkout block is the default checkout (a block theme's checkout template or the checkout page contains `woocommerce/checkout`). Each enabled location becomes a rate `pickup_location:{index}` with meta `pickup_location` (name), `pickup_address`, `pickup_details` and hidden `_pickup_location_address` (used as the order's tax location).

In the Checkout block, pickup is a separate "Pickup" toggle, not a delivery rate:

- A method is a pickup method when its `$supports` contains `local-pickup` (plus core `local_pickup`, and the legacy filter `woocommerce_local_pickup_methods`). These ids reach the page as `collectableMethodIds`.
- If any package lacks a pickup rate, pickup rates are removed from **all** packages and the toggle disappears.
- With "Hide shipping costs until an address is entered", the block checkout keeps only pickup rates until there is an address.
- Pickup orders are taxed at the store base address by default; filter `woocommerce_apply_base_tax_for_local_pickup` (return `false` to disable).
- Pickup methods are still calculated for non-shippable destinations.

A custom pickup-point method:

```php
class My_Plugin_Pickup_Points_Method extends WC_Shipping_Method {

	public function __construct( $instance_id = 0 ) {
		parent::__construct( $instance_id );
		$this->id                   = 'my_plugin_pickup_points';
		$this->method_title         = __( 'Pickup points', 'my-plugin' );
		$this->supports             = array( 'shipping-zones', 'instance-settings', 'instance-settings-modal', 'local-pickup' );
		$this->instance_form_fields = array(
			'title' => array(
				'title'   => __( 'Name', 'my-plugin' ),
				'type'    => 'text',
				'default' => __( 'Pickup', 'my-plugin' ),
			),
		);
		$this->title = $this->get_option( 'title' );
		add_action( 'woocommerce_update_options_shipping_' . $this->id, array( $this, 'process_admin_options' ) );
	}

	public function calculate_shipping( $package = array() ) {
		$points = array(
			'store' => array( 'name' => __( 'Main store', 'my-plugin' ), 'address' => 'Av. Example 123, Santiago', 'details' => __( 'Mon to Fri, 10:00 to 18:00', 'my-plugin' ) ),
		);
		foreach ( $points as $slug => $point ) {
			$this->add_rate(
				array(
					'id'        => $this->get_rate_id( $slug ),
					'label'     => $this->title . ' (' . $point['name'] . ')',
					'cost'      => 0,
					'package'   => $package,
					'meta_data' => array(
						'pickup_location' => $point['name'],    // Same keys as core, read by the block UI.
						'pickup_address'  => $point['address'],
						'pickup_details'  => $point['details'],
					),
				)
			);
		}
	}
}
```

## How rates reach the browser

`GET /wc/store/v1/cart` (and every cart-mutating response) includes:

```json
"shipping_rates": [ {
  "package_id": 0,
  "name": "Shipment",
  "destination": { "address_1": "…", "address_2": "", "city": "Santiago", "state": "CL-RM", "postcode": "", "country": "CL" },
  "items": [ { "key": "…", "name": "…", "quantity": 2 } ],
  "shipping_rates": [ {
    "rate_id": "my_plugin_courier:3",
    "name": "Home delivery",
    "description": "Tracked delivery to your door.",
    "delivery_time": "2 to 4 business days",
    "price": "5970", "taxes": "0",
    "instance_id": 3, "method_id": "my_plugin_courier",
    "meta_data": [ { "key": "Weight", "value": "2.50 kg" }, { "key": "Items", "value": "…" } ],
    "selected": true,
    "currency_code": "CLP", "currency_minor_unit": 0
  } ]
} ]
```

- `price` and `taxes` are integer strings in minor units of the store decimals.
- `meta_data` excludes `_`-prefixed keys: hidden meta is safe for internal ids, but never put secrets in visible meta.
- The block's rate list shows `delivery_time` next to the price; selecting a rate calls `POST /cart/select-shipping-rate`, and address edits call `POST /cart/update-customer`, which recalculates.
- Since checkout and cart are never cached and the Mini-Cart reads `/cart` in the browser, nothing about shipping belongs in cached HTML. A "free shipping from $X" bar on a cached page must compute progress client-side from the Store API cart (see `cache-and-store-api.md`).

## Common mistakes

- Defining the class at plugin load instead of on `woocommerce_shipping_init`, or registering it under a key different from `$id`.
- No `woocommerce_update_options_shipping_{id}` hook: instance settings never save.
- Missing `instance-settings-modal`: the zone UI cannot open the settings in a modal.
- Calling `get_option()` before `$instance_form_fields` is set: instance values are ignored.
- No instance `title` field: every zone shows the same label.
- Remote calls without a timeout or cache inside `calculate_shipping()`.
- `'description'` or `'delivery_time'` passed to `add_rate()`: ignored. Set them on the `WC_Shipping_Rate`.
- Float costs for zero-decimal currencies without `price_decimals`: no rounding happens.
- Reimplementing "hide other rates when free" in `woocommerce_package_rates` (and often inverting it); use the core option.
- A pickup method without `local-pickup` in `$supports`: it appears as a delivery rate and never under the Pickup toggle.
- Expecting `pickup_location` on a classic-checkout site: it only registers when the Checkout block is the default checkout.
- Debugging rate changes without debug mode: the session cache returns old rates.
