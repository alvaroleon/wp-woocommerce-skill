# Checkout block: fields, Store API checkout, JS extensibility

The Checkout and Cart blocks are React apps on the Store API (`/wc/store/v1/checkout`, `/cart`). None of the classic checkout hooks run there. This file covers adding fields with the Additional Checkout Fields API, the server-side checkout sequence and its hooks, rate limiting, and extending the blocks from JS. Verified against the WooCommerce 11.2.1 source; anything not verified is marked **untested**. Cache rules, the nonce and ExtendSchema basics live in `cache-and-store-api.md`.

## Additional Checkout Fields API

Use it for any plain input: text, select, checkbox, date. The field renders in the Checkout block, is validated in the browser and on the server, is stored on the order (and the customer for some locations), and shows up in the admin order screen, emails, order confirmation and My Account without more code.

### Registration

Call `woocommerce_register_additional_checkout_field()` on `woocommerce_init`. Since 11.0, registering before `after_setup_theme` triggers `_doing_it_wrong` (translations are not ready yet). Invalid options are reported with `_doing_it_wrong` and the field is skipped; nothing throws.

```php
add_action( 'woocommerce_init', 'my_plugin_register_checkout_fields' );

function my_plugin_register_checkout_fields() {
	if ( ! function_exists( 'woocommerce_register_additional_checkout_field' ) ) {
		return;
	}

	// Address location: rendered in billing AND shipping forms, stored per group.
	woocommerce_register_additional_checkout_field(
		array(
			'id'                => 'my-plugin/tax-id',
			'label'             => __( 'Tax ID', 'my-plugin' ),
			'optionalLabel'     => __( 'Tax ID (optional)', 'my-plugin' ),
			'location'          => 'address',
			'type'              => 'text',
			'required'          => false,
			'attributes'        => array(
				'autocomplete'     => 'off',
				'maxLength'        => 12,
				'pattern'          => '[0-9kK.\-]+',
				'title'            => __( 'Digits, dots, dash and K only.', 'my-plugin' ),
				'aria-describedby' => 'my-plugin-tax-id-help',
				'data-my-plugin'   => 'tax-id',
			),
			'sanitize_callback' => 'my_plugin_sanitize_tax_id',
			'validate_callback' => 'my_plugin_validate_tax_id',
		)
	);

	// Contact location: stored on the order AND the customer (editable in My Account > Account details).
	woocommerce_register_additional_checkout_field(
		array(
			'id'            => 'my-plugin/legal-age',
			'label'         => __( 'I confirm I am of legal drinking age', 'my-plugin' ),
			'location'      => 'contact',
			'type'          => 'checkbox',
			'required'      => true,
			'error_message' => __( 'You must be of legal age to buy wine.', 'my-plugin' ), // Checkbox only.
		)
	);

	// Order location: stored on the order only.
	woocommerce_register_additional_checkout_field(
		array(
			'id'          => 'my-plugin/how-did-you-hear',
			'label'       => __( 'How did you hear about us?', 'my-plugin' ),
			'location'    => 'order',
			'type'        => 'select',
			'placeholder' => __( 'Choose an option', 'my-plugin' ),
			'options'     => array(
				array(
					'value' => 'instagram',
					'label' => __( 'Instagram', 'my-plugin' ),
				),
				array(
					'value' => 'friend',
					'label' => __( 'A friend', 'my-plugin' ),
				),
			),
		)
	);

	// Date (11.2+): YYYY-MM-DD; min/max absolute or ISO 8601-2 duration in whole days.
	woocommerce_register_additional_checkout_field(
		array(
			'id'                         => 'my-plugin/delivery-date',
			'label'                      => __( 'Preferred delivery date', 'my-plugin' ),
			'location'                   => 'order',
			'type'                       => 'date',
			'required'                   => true,
			'min'                        => 'P2D', // Today + 2 days, resolved at validation time in the store timezone.
			'max'                        => 'P2M',
			'show_in_order_confirmation' => true,
		)
	);
}

function my_plugin_sanitize_tax_id( $value, array $field ) {
	return strtoupper( preg_replace( '/[^0-9kK]/', '', (string) $value ) );
}

function my_plugin_validate_tax_id( $value, array $field ) {
	if ( '' !== $value && ! preg_match( '/^[0-9]{7,8}[0-9K]$/', $value ) ) {
		return new WP_Error( 'my_plugin_invalid_tax_id', __( 'Please enter a valid tax ID.', 'my-plugin' ) );
	}
	return true; // Returning false instead yields the generic "Please provide a valid %s".
}
```

### Arguments

| Arg | Notes |
|---|---|
| `id` | Required, `namespace/name`. Without a `/` the field is rejected. Used in meta keys and the Store API payload. |
| `label` | Required. `optionalLabel` defaults to "%s (optional)". |
| `location` | `contact`, `address` or `order` (`additional` is a deprecated alias of `order`). |
| `type` | `text` (default), `select`, `checkbox`, `date` (11.2). Anything else is rejected. |
| `required` | `true`/`false` or a JSON Schema rule (below). Hidden fields are never required. |
| `hidden` | JSON Schema rule only. `hidden => true` is not supported: the field is registered visible with a notice. |
| `validation` | JSON Schema applied to the field value in context. On failure the server answers a generic "Invalid field."; use `validate_callback` for a custom message. |
| `attributes` | Only `maxLength`, `readOnly`, `pattern`, `autocomplete`, `autocapitalize`, `title`, plus `aria-*` and `data-*`. Others dropped with a notice; values are `esc_attr`'d. |
| `sanitize_callback` / `validate_callback` | `( $value, $field )`. Defaults come from the type class. Validate returns `true`, `false` or `WP_Error`. |
| `show_in_order_confirmation` | Default `true`. |
| `options`, `placeholder` | Select only. Each option needs `value` and `label`; duplicates are dropped. The REST schema gets an `enum`. |
| `error_message` | Checkbox only, used when a required checkbox is unchecked. Stored as `'1'`/`'0'`, read back as bool. |
| `min`, `max` | Date only (11.2). `YYYY-MM-DD` or a signed ISO 8601-2 duration (`P2D`, `-P18Y`), whole days only. Type-level date validation (format, min, max) always runs, even with a custom `validate_callback`. |

### Conditional `required`, `hidden` and `validation`

Rules are JSON Schema draft-07 evaluated against a document object, in the browser and again on the server. The document has `cart` (`coupons`, `shipping_rates` as `method_id:instance_id`, `items` as product ids, `items_type`, `items_count`, `items_weight`, `needs_shipping`, `prefers_collection`, `totals.total_price`, `totals.total_tax`, `extensions`), `checkout` (`create_account`, `customer_note`, `additional_fields`, `payment_method`) and `customer` (`id`, `billing_address`, `shipping_address`, `additional_fields`, `address`). Address objects carry `first_name` … `country`, `email`, `phone`.

```php
woocommerce_register_additional_checkout_field(
	array(
		'id'       => 'my-plugin/company-tax-id',
		'label'    => __( 'Company tax ID', 'my-plugin' ),
		'location' => 'order',
		'type'     => 'text',
		// Required when shipping to Chile.
		'required' => array(
			'customer' => array(
				'properties' => array(
					'shipping_address' => array(
						'properties' => array( 'country' => array( 'const' => 'CL' ) ),
					),
				),
			),
		),
		// Hidden for local pickup.
		'hidden'   => array(
			'cart' => array(
				'properties' => array( 'prefers_collection' => array( 'const' => true ) ),
			),
		),
	)
);
```

A top-level rule with `cart`, `checkout` or `customer` keys is wrapped as an object schema for you. For date fields, `formatMinimum`/`formatMaximum`/`formatExclusiveMinimum`/`formatExclusiveMaximum` in `validation` get `type: string, format: date` added automatically.

### Validation layers (server)

1. Type-level checks (date format and range) — cannot be bypassed by a custom callback. Select values are constrained by the `enum` in the REST schema.
2. The field's `validate_callback`.
3. Action `woocommerce_validate_additional_field( WP_Error $errors, string $field_key, $value )`.
4. Action `woocommerce_blocks_validate_location_{address|contact|order}_fields( WP_Error $errors, array $fields, string $group )` for cross-field checks; for `address` it fires once per group (`billing`, `shipping`).

Exceptions thrown inside sanitize/validate callbacks and these actions are caught and turned into a PHP warning, and checkout continues. Report problems through `WP_Error`, never by throwing.

```php
add_action( 'woocommerce_blocks_validate_location_order_fields', 'my_plugin_validate_order_fields', 10, 3 );

function my_plugin_validate_order_fields( WP_Error $errors, array $fields, $group ) {
	$date = isset( $fields['my-plugin/delivery-date'] ) ? $fields['my-plugin/delivery-date'] : '';
	if ( $date && 7 === (int) wp_date( 'N', strtotime( $date . ' 12:00:00' ) ) ) {
		$errors->add( 'my_plugin_no_sunday', __( 'We do not deliver on Sundays.', 'my-plugin' ) );
	}
}
```

Filter `woocommerce_sanitize_additional_field( $value, $field_key )` runs after the field's own sanitizer.

### Reading values (HPOS-safe)

Use the `CheckoutFields` service; don't hard-code meta keys (`_wc_billing/<id>`, `_wc_shipping/<id>`, `_wc_other/<id>`). Group is `other` for contact and order fields, `billing`/`shipping` for address fields. Works with `WC_Order` and `WC_Customer`.

```php
use Automattic\WooCommerce\Blocks\Domain\Services\CheckoutFields;
use Automattic\WooCommerce\Blocks\Package;

function my_plugin_get_delivery_date( WC_Order $order ) {
	$fields = Package::container()->get( CheckoutFields::class );
	return (string) $fields->get_field_from_object( 'my-plugin/delivery-date', $order, 'other' );
}

function my_plugin_get_billing_tax_id( WC_Order $order ) {
	$fields = Package::container()->get( CheckoutFields::class );
	return (string) $fields->get_field_from_object( 'my-plugin/tax-id', $order, 'billing' );
}
```

- `get_all_fields_from_object( $object, $group = 'other', $all = false )` returns `[ id => value ]`.
- `get_order_additional_fields_with_values( $order, $location, $group, 'view' )` returns definitions plus display-formatted values (date in store format, select label), and returns `[]` unless `$order->get_created_via() === 'store-api'`: values exist only for orders placed through the Checkout block / Store API.
- Inside `woocommerce_store_api_checkout_update_order_from_request` the values are already persisted on the order.
- Mirror to a legacy meta key: action `woocommerce_set_additional_field_value( $key, $value, $group, $wc_object )`. Pull a default from legacy data: filter `woocommerce_get_default_value_for_{$key}( null, $group, $wc_object )`.

### Where they display

Order confirmation blocks, customer emails (`woocommerce_email_customer_details` for contact/order, the address section for address fields), the admin order screen (editable), and My Account (order details, addresses, Account details for contact fields). Hide a field from a given context with filter `woocommerce_filter_fields_for_order_confirmation( $show, $field, $fields, $context, $checkout_fields )`; `$context['caller']` identifies emails (`WC_Email::additional_checkout_fields`) and carries `sent_to_admin`/`plain_text`.

### Fields API or ExtendSchema?

Plain inputs → Additional Checkout Fields. Custom UI (availability calendar, gift-message composer, multi-value widgets) or non-field data → an inner block posting `extensions` data validated by ExtendSchema (see JS section).

## The Store API checkout sequence

### Form interactions: `PUT`/`PATCH /checkout`

The block sends updates while the shopper fills the form (with `__experimental_calc_totals=true` when payment method, notes or additional fields change). Draft order creation is deferred to place-order time (source `@since 10.8.0`; announced in the 10.9 changelog): with no pending/failed order in the session, the request only writes the session (payment method, notes, additional fields) and fires `woocommerce_store_api_checkout_update_draft( WP_REST_Request $request )`. No `WC_Order` exists there: read `WC()->cart`, `WC()->customer`, the request, and keep your own state in `WC()->session`. If a failed/pending order is in the session (payment retry), the request updates that order instead.

### Place order: `POST /checkout`

In this order (`Checkout::process_order()`):

1. Additional fields validation (`validate_callback` and the layers above) and ExtendSchema `extensions` validation.
2. Store settings allow this user to place the order; cart not empty.
3. Customer updated from the request → `woocommerce_store_api_checkout_update_customer_from_request( WC_Customer $customer, WP_REST_Request $request )`.
4. `calculate_totals()`, then `validate_cart()` (fires `woocommerce_store_api_validate_cart_item` and `woocommerce_store_api_cart_errors`).
5. `expected_total` check (below).
6. Order created from the cart → `woocommerce_store_api_checkout_order_created( WC_Order $order )` (once, new orders only), then `woocommerce_store_api_checkout_update_order_meta( WC_Order $order )`.
7. Notes, payment method and additional fields set → `woocommerce_store_api_checkout_update_order_from_request( WC_Order $order, WP_REST_Request $request )`. On POST the order is not saved here; core saves it later in the request.
8. Customer created/persisted (guest → account if requested).
9. Address/email validation, then `woocommerce_checkout_validate_order_before_payment( WC_Order $order, WP_Error $errors )` (9.9). Errors become a 400 `woocommerce_rest_checkout_custom_validation_error`.
10. Coupons held, stock reserved, status `pending` → `woocommerce_store_api_checkout_order_processed( WC_Order $order )`.
11. Payment via `woocommerce_rest_checkout_process_payment_with_context` (see `payments.md`).

**`expected_total` (11.1):** the Checkout block sends `expected_total` (cart `totals.total_price`, digits in minor units) on POST. If the server's recalculated total is **higher**, checkout fails with `409 woocommerce_rest_checkout_total_mismatch` and the refreshed cart, before any order exists. A lower total passes. Clients that cannot know the total (some express payments) omit it and skip the check.

### Errors

- `RouteException( $error_code, $message, $http_status = 400, $additional_data = array() )` is the clean way to stop a request with a message the block shows.
- `InvalidCartException` (409) comes from errors added in `woocommerce_store_api_cart_errors`.
- Any other `\Exception` reaching the checkout route becomes a 500 `woocommerce_rest_unknown_server_error`. Throwing in `woocommerce_store_api_checkout_update_order_meta` puts the block in an error state and stops checkout; prefer `RouteException`.

```php
use Automattic\WooCommerce\StoreApi\Exceptions\RouteException;

add_action( 'woocommerce_checkout_validate_order_before_payment', 'my_plugin_check_delivery_zone', 10, 2 );
add_action( 'woocommerce_store_api_checkout_update_order_from_request', 'my_plugin_save_gift_message', 10, 2 );
add_action( 'woocommerce_store_api_checkout_order_processed', 'my_plugin_enqueue_order_sync' );

function my_plugin_check_delivery_zone( WC_Order $order, WP_Error $errors ) {
	$blocked = (array) get_option( 'my_plugin_blocked_cities', array() );
	if ( in_array( strtolower( $order->get_shipping_city() ), $blocked, true ) ) {
		$errors->add( 'my_plugin_no_delivery', __( 'We cannot deliver to that city yet.', 'my-plugin' ) );
	}
}

function my_plugin_save_gift_message( WC_Order $order, WP_REST_Request $request ) {
	$extensions = $request->get_param( 'extensions' );
	$message    = isset( $extensions['my-plugin']['gift_message'] ) ? $extensions['my-plugin']['gift_message'] : '';
	if ( '' !== $message ) {
		$order->update_meta_data( '_my_plugin_gift_message', $message ); // Already sanitized by the schema. No save() here.
	}
}

function my_plugin_enqueue_order_sync( WC_Order $order ) {
	// Never call a third party inline here: enqueue a unique Action Scheduler job.
	as_enqueue_async_action( 'my_plugin_sync_order', array( $order->get_id() ), 'my-plugin', true );
}
```

### Other server-side helpers

- `woocommerce_store_api_register_payment_requirements( array( 'data_callback' => $callable ) )`: the callable returns requirement strings that cart payment methods must support.
- `woocommerce_store_api_get_formatter( 'money' )->format( $amount, array( 'decimals' => wc_get_price_decimals() ) )`: the same minor-unit string format as cart totals.

## Rate limiting

- Generic Store API rate limiting is off by default (limit 25 requests / 10 s when enabled). Filter `woocommerce_store_api_rate_limit_options` returning `enabled`, `proxy_support`, `limit`, `seconds`. Exceeding it answers 429 with `RateLimit-*` headers and fires `woocommerce_store_api_rate_limit_exceeded( $ip, $action_id )`. Identifier: user id or `md5( IP )`, filter `woocommerce_store_api_rate_limit_id` (9.8).
- Feature **Rate limit Checkout** (`rate_limit_checkout`, WooCommerce › Settings › Advanced › Features, off by default) limits `POST /checkout` to 3 per 60 s by adding a priority-1 filter on the options; a later filter of yours overrides it.
- `proxy_support => true` reads `X-Real-IP`, `Client-IP`, `X-Forwarded-For`, `Forwarded` in that order. Those are client-controlled unless your proxy overwrites them; enable it only behind a proxy that does.

```php
add_filter( 'woocommerce_store_api_rate_limit_options', 'my_plugin_store_api_rate_limits', 20 );

function my_plugin_store_api_rate_limits( array $options ) {
	$options['enabled']       = true;
	$options['proxy_support'] = true; // Only behind a proxy that sets X-Real-IP / X-Forwarded-For itself.
	return $options;
}
```

## Extending the blocks from JS

### Build

`@woocommerce/blocks-checkout`, `@woocommerce/block-data` and `@woocommerce/blocks-registry` are not on npm; `@woocommerce/settings` on npm is a stale 2019 package. They exist only as WordPress script externals, so build with `@wordpress/scripts` plus `@woocommerce/dependency-extraction-webpack-plugin`, which maps the imports to globals and writes the handles into `*.asset.php`. It only maps classic scripts, not script modules. Scaffold alternative: `npx @wordpress/create-block -t @woocommerce/extend-cart-checkout-block my-plugin` (template contents **untested** here).

```json
{
	"name": "my-plugin",
	"private": true,
	"scripts": { "build": "wp-scripts build", "start": "wp-scripts start" },
	"devDependencies": {
		"@woocommerce/dependency-extraction-webpack-plugin": "^5.1.0",
		"@wordpress/scripts": "^37.0.0"
	}
}
```

```js
// webpack.config.js
const defaultConfig = require( '@wordpress/scripts/config/webpack.config' );
const WooCommerceDependencyExtractionWebpackPlugin = require( '@woocommerce/dependency-extraction-webpack-plugin' );

module.exports = {
	...defaultConfig,
	entry: {
		'gift-message/frontend': './src/gift-message/frontend.js',
		'gift-message/editor': './src/gift-message/editor.js',
	},
	plugins: [
		...defaultConfig.plugins.filter( ( p ) => p.constructor.name !== 'DependencyExtractionWebpackPlugin' ),
		new WooCommerceDependencyExtractionWebpackPlugin(),
	],
};
```

| Import | Global | Script handle |
|---|---|---|
| `@woocommerce/blocks-checkout` | `wc.blocksCheckout` | `wc-blocks-checkout` |
| `@woocommerce/block-data` | `wc.wcBlocksData` | `wc-blocks-data-store` |
| `@woocommerce/blocks-registry` | `wc.wcBlocksRegistry` | `wc-blocks-registry` |
| `@woocommerce/settings` | `wc.wcSettings` | `wc-settings` |
| `@woocommerce/price-format` | `wc.priceFormat` | `wc-price-format` |
| `@woocommerce/blocks-components` | `wc.blocksComponents` | `wc-blocks-components` |

### Loading scripts: `IntegrationInterface`

Register an integration per block on `woocommerce_blocks_{block}_block_registration` (`checkout`, `cart`, `mini-cart`). Its scripts load wherever that block renders; `get_script_data()` is exposed as `getSetting( '<name>_data' )` and printed inline, so it must be the same for every visitor. Since 11.1 WooCommerce skips block registration on REST/Store API requests, so these hooks fire only on rendering requests.

```php
use Automattic\WooCommerce\Blocks\Integrations\IntegrationInterface;

class My_Plugin_Checkout_Integration implements IntegrationInterface {
	public function get_name() {
		return 'my-plugin'; // getSetting( 'my-plugin_data' ).
	}

	public function initialize() {
		foreach ( array( 'frontend', 'editor' ) as $part ) {
			$asset = include plugin_dir_path( __FILE__ ) . "build/gift-message/{$part}.asset.php";
			wp_register_script(
				"my-plugin-gift-message-{$part}",
				plugins_url( "build/gift-message/{$part}.js", __FILE__ ),
				$asset['dependencies'],
				$asset['version'],
				true
			);
			wp_set_script_translations( "my-plugin-gift-message-{$part}", 'my-plugin' );
		}
	}

	public function get_script_handles() {
		return array( 'my-plugin-gift-message-frontend' );
	}

	public function get_editor_script_handles() {
		return array( 'my-plugin-gift-message-editor' );
	}

	public function get_script_data() {
		return array( 'maxLength' => 200 ); // Site-wide only.
	}
}

add_action(
	'woocommerce_blocks_checkout_block_registration',
	static function ( $registry ) {
		$registry->register( new My_Plugin_Checkout_Integration() );
	}
);
```

### An inner block in a checkout area

The block needs three registrations: PHP (so WooCommerce adds `data-block-name` and `data-*` attributes to its saved markup on render — it does this only for blocks registered server-side whose `parent` is a `woocommerce/*` block), the editor (`registerBlockType`), and the frontend (`registerCheckoutBlock`, which mounts your component where that markup is). A `lock` attribute defaulting to `{ "remove": true }` makes the parent's forced layout insert the block automatically when the merchant opens the checkout in the editor.

```json
{
	"$schema": "https://schemas.wp.org/trunk/block.json",
	"apiVersion": 3,
	"name": "my-plugin/gift-message",
	"title": "Gift message",
	"category": "woocommerce",
	"parent": [ "woocommerce/checkout-fields-block" ],
	"attributes": {
		"lock": { "type": "object", "default": { "remove": true, "move": true } }
	},
	"supports": { "html": false, "multiple": false, "reusable": false },
	"textdomain": "my-plugin"
}
```

```php
add_action( 'init', 'my_plugin_register_gift_message_block' );

function my_plugin_register_gift_message_block() {
	// @wordpress/scripts copies src/**/block.json into build/. Register on init, before the first render.
	register_block_type( plugin_dir_path( __FILE__ ) . 'build/gift-message' );
}
```

```js
// src/gift-message/editor.js
import { registerBlockType } from '@wordpress/blocks';
import { useBlockProps } from '@wordpress/block-editor';
import { __ } from '@wordpress/i18n';
import metadata from './block.json';

registerBlockType( metadata, {
	edit: () => (
		<div { ...useBlockProps() }>
			<textarea disabled placeholder={ __( 'Gift message (shoppers write it here)', 'my-plugin' ) } />
		</div>
	),
	save: () => <div { ...useBlockProps.save() } />,
} );
```

```js
// src/gift-message/frontend.js
import { registerCheckoutBlock } from '@woocommerce/blocks-checkout';
import { validationStore } from '@woocommerce/block-data';
import { getSetting } from '@woocommerce/settings';
import { useDispatch } from '@wordpress/data';
import { useEffect, useState } from '@wordpress/element';
import { __ } from '@wordpress/i18n';
import metadata from './block.json';

const { maxLength = 200 } = getSetting( 'my-plugin_data', {} );
const ERROR_ID = 'my-plugin-gift-message';

const GiftMessage = ( { checkoutExtensionData } ) => {
	const { setExtensionData } = checkoutExtensionData; // ( namespace, key, value ), merges into extensions.
	const { setValidationErrors, clearValidationError } = useDispatch( validationStore );
	const [ message, setMessage ] = useState( '' );

	useEffect( () => {
		setExtensionData( 'my-plugin', 'gift_message', message );
		if ( message.length > maxLength ) {
			// Any error in the validation store blocks Place Order.
			setValidationErrors( {
				[ ERROR_ID ]: { message: __( 'Your gift message is too long.', 'my-plugin' ), hidden: false },
			} );
		} else {
			clearValidationError( ERROR_ID );
		}
	}, [ message, setExtensionData, setValidationErrors, clearValidationError ] );

	return (
		<div className="wc-block-components-text-input is-active">
			<label htmlFor="my-plugin-gift-message">{ __( 'Gift message (optional)', 'my-plugin' ) }</label>
			<textarea
				id="my-plugin-gift-message"
				value={ message }
				onChange={ ( event ) => setMessage( event.target.value ) }
			/>
		</div>
	);
};

registerCheckoutBlock( { metadata, component: GiftMessage } ); // component may also be React.lazy().
```

The component receives `checkoutExtensionData`, `extensions`, `cart` and `validation` from the parent. `registerCheckoutBlock` rejects the registration unless `metadata.name` is set and `metadata.parent` includes a valid area.

Valid areas (`innerBlockAreas`):

- Checkout: `woocommerce/checkout-fields-block`, `checkout-totals-block`, `checkout-contact-information-block`, `checkout-shipping-address-block`, `checkout-billing-address-block`, `checkout-shipping-method-block`, `checkout-shipping-methods-block`, `checkout-pickup-options-block`, `checkout-payment-block`, `checkout-order-summary-block`, `checkout-order-summary-totals-block` (all `woocommerce/`).
- Cart: `woocommerce/cart`, `filled-cart-block`, `empty-cart-block`, `cart-items-block`, `cart-cross-sells-block`, `cart-totals-block`, `cart-order-summary-block`, `cart-order-summary-totals-block`.
- Mini-Cart areas are still listed, but in 11.x the Mini-Cart renders in PHP/Interactivity API, so a React inner block there will not render on the frontend (**untested**, inferred from source).

### Accepting the posted data: ExtendSchema on `checkout`

The Checkout block posts `extensions: { "my-plugin": { "gift_message": "…" } }` with `POST /checkout`. Declare it, or it is dropped. Each property runs through `arg_options.validate_callback`/`sanitize_callback` (`( $value, $request, $param )`) or the WP schema validators.

```php
use Automattic\WooCommerce\StoreApi\Schemas\V1\CheckoutSchema;

add_action( 'woocommerce_blocks_loaded', 'my_plugin_extend_checkout_schema' );

function my_plugin_extend_checkout_schema() {
	woocommerce_store_api_register_endpoint_data(
		array(
			'endpoint'        => CheckoutSchema::IDENTIFIER,
			'namespace'       => 'my-plugin',
			'schema_type'     => ARRAY_A,
			'data_callback'   => static function () {
				return array( 'gift_message' => '' );
			},
			'schema_callback' => static function () {
				return array(
					'gift_message' => array(
						'description' => __( 'Gift message.', 'my-plugin' ),
						'type'        => array( 'string', 'null' ), // Empty values skip validation.
						'context'     => array( 'view', 'edit' ),
						'arg_options' => array(
							'validate_callback' => static function ( $value ) {
								return mb_strlen( (string) $value ) <= 200
									? true
									: new WP_Error( 'my_plugin_gift_too_long', __( 'Your gift message is too long.', 'my-plugin' ) );
							},
							'sanitize_callback' => static function ( $value ) {
								return sanitize_textarea_field( (string) $value );
							},
						),
					),
				);
			},
		)
	);
}
```

`my_plugin_save_gift_message()` in the checkout section stores it on the order.

### Checkout filters

```js
import { registerCheckoutFilters } from '@woocommerce/blocks-checkout';
import { __ } from '@wordpress/i18n';

registerCheckoutFilters( 'my-plugin', {
	placeOrderButtonLabel: () => __( 'Pay now', 'my-plugin' ),
	itemName: ( name, extensions ) =>
		extensions?.[ 'my-plugin' ]?.bottles > 1 ? `${ name } (${ extensions[ 'my-plugin' ].bottles })` : name,
	showRemoveItemLink: ( show, extensions, { cartItem } ) => show && ! cartItem?.extensions?.[ 'my-plugin' ]?.locked,
} );
```

Filter names in 11.2.1: `additionalCartCheckoutInnerBlockTypes`, `cartItemClass`, `cartItemPrice` and `subtotalPriceFormat`/`saleBadgePriceFormat` (keep the `<price/>` placeholder), `cartItemScreenReaderPrice`, `coupons`, `itemName`, `placeOrderButtonLabel`, `proceedToCheckoutButtonLabel`, `proceedToCheckoutButtonLink`, `showApplyCouponNotice`, `showRemoveCouponNotice`, `showRemoveItemLink`, `totalLabel`, `totalValue`. Callbacks get `( value, extensions, args )`.

### Slot fills

Fills render inside both Cart and Checkout through plugin scope `woocommerce-checkout`. Slots: `ExperimentalOrderMeta`, `ExperimentalOrderShippingPackages`, `ExperimentalOrderLocalPickupPackages`, `ExperimentalDiscountsMeta`; fill props `{ cart, extensions, context }` with `context` `'woocommerce/cart'` or `'woocommerce/checkout'`.

```js
import { registerPlugin } from '@wordpress/plugins';
import { ExperimentalOrderMeta } from '@woocommerce/blocks-checkout';
import { __, sprintf } from '@wordpress/i18n';

const BottleCount = ( { cart } ) => {
	const bottles = cart.cartItems.reduce( ( sum, item ) => sum + ( item.extensions?.[ 'my-plugin' ]?.bottles || 0 ), 0 );
	/* translators: %d: number of bottles. */
	return bottles ? <p className="my-plugin-bottles">{ sprintf( __( 'Bottles in your order: %d', 'my-plugin' ), bottles ) }</p> : null;
};

registerPlugin( 'my-plugin-order-meta', {
	scope: 'woocommerce-checkout',
	render: () => <ExperimentalOrderMeta><BottleCount /></ExperimentalOrderMeta>,
} );
```

### Data stores (`@woocommerce/block-data`)

Stores: `cartStore` (`wc/store/cart`), `checkoutStore` (`wc/store/checkout`), `paymentStore` (`wc/store/payment`), `validationStore` (`wc/store/validation`), plus `storeNoticesStore`, `collectionsStore`, `queryStateStore`, `schemaStore`.

- Cart selectors: `getCartData`, `getCartTotals`, `getCartErrors`, `getCartMeta`, `getCustomerData`, `getShippingRates`, `getNeedsShipping`, `isApplyingCoupon`. Action `applyExtensionCartUpdate` (behind `extensionCartUpdate`).
- Checkout selectors: `getExtensionData`, `getAdditionalFields`, `getOrderId`, `getCustomerId`, `getCheckoutStatus`, `prefersCollection`, `isProcessing`, `isComplete`, `hasError`, `getRedirectUrl`, `getShouldCreateAccount`, `getUseShippingAsBilling`. Actions: `setExtensionData( namespace, data, replace = false )`, `setAdditionalFields`, `setPrefersCollection`, `disableCheckoutFor`.
- Validation: `setValidationErrors`, `clearValidationError(s)`, `clearAllValidationErrors`, `hideValidationError`, `showValidationError`, `showAllValidationErrors`; selectors `getValidationError`, `hasValidationErrors`.

```js
import { useSelect } from '@wordpress/data';
import { cartStore } from '@woocommerce/block-data';

const useCartTotal = () => useSelect( ( select ) => select( cartStore ).getCartTotals().total_price, [] );
```

### `extensionCartUpdate`

```js
import { extensionCartUpdate } from '@woocommerce/blocks-checkout';

// POST /wc/store/v1/cart/extensions → your woocommerce_store_api_register_update_callback → totals recalculated → wc/store/cart updated.
await extensionCartUpdate( { namespace: 'my-plugin-gift-wrap', data: { enabled: true } } );
```

Pass `overwriteDirtyCustomerData: true` to let the response replace address fields the shopper is editing. Server side and fees: `cache-and-store-api.md`.

## Classic → blocks mapping

| Classic (don't use) | Blocks / Store API |
|---|---|
| `woocommerce_checkout_fields`, `woocommerce_after_order_notes` + `woocommerce_form_field()` | `woocommerce_register_additional_checkout_field()` |
| `woocommerce_checkout_process`, `woocommerce_after_checkout_validation` | Field `validate_callback`, `woocommerce_validate_additional_field`, `woocommerce_blocks_validate_location_*_fields`, `woocommerce_checkout_validate_order_before_payment` |
| `woocommerce_checkout_update_order_meta`, `woocommerce_checkout_create_order` | `woocommerce_store_api_checkout_update_order_from_request( $order, $request )` |
| `woocommerce_checkout_order_processed` | `woocommerce_store_api_checkout_order_processed( $order )` |
| `woocommerce_checkout_update_order_review` (live form changes) | `woocommerce_store_api_checkout_update_draft( $request )` |
| `$order->get_meta( '_my_field' )` for a checkout field | `CheckoutFields::get_field_from_object()` |
| `woocommerce_add_to_cart_validation` | `woocommerce_store_api_validate_add_to_cart` (throw `RouteException`) |
| `woocommerce_update_cart_validation` | `woocommerce_store_api_cart_item_quantity_validation` (11.2, return `WP_Error`) |
| `wc_add_notice()` errors | `RouteException`, `WP_Error` in the hooks above, `woocommerce_store_api_cart_errors` |
| `checkout_error` / `updated_checkout` jQuery events | `wc/store/checkout` and `wc/store/validation` stores, `registerCheckoutFilters`, slot fills |
| Custom AJAX to change the cart | `woocommerce_store_api_register_update_callback` + `extensionCartUpdate` |
| `[woocommerce_checkout]` shortcode | Checkout block (Additional Checkout Fields never render in the shortcode) |

## Common mistakes

- Registering fields on `plugins_loaded` or at file load instead of `woocommerce_init`.
- Throwing from `validate_callback`, `sanitize_callback` or the validation actions: the exception is swallowed with a warning and checkout proceeds. Return `WP_Error`.
- `hidden => true` to hide a field; only rule-based hiding exists.
- Reading `_wc_other/...` meta directly, or expecting additional-field values on orders not created via `store-api`.
- Using `woocommerce_store_api_register_update_callback` as a checkout validator; it only serves `POST /cart/extensions`.
- Expecting a `WC_Order` in `woocommerce_store_api_checkout_update_draft`, or relying on a draft order existing before Place Order.
- Calling `$order->save()` in `woocommerce_store_api_checkout_update_order_from_request` on POST (core saves it), or calling remote APIs in any checkout hook instead of enqueueing an Action Scheduler job.
- Plain `\Exception` instead of `RouteException` (500 instead of a readable error).
- Posting `extensions` data without declaring it with ExtendSchema on `checkout`: it is silently dropped.
- Installing `@woocommerce/blocks-checkout` or `@woocommerce/settings` from npm, or building without the WooCommerce dependency extraction plugin.
- Registering the inner block only in JS: without the PHP `register_block_type()` the frontend never gets `data-block-name` and the component never mounts.
- Per-user data in `IntegrationInterface::get_script_data()`.
- `proxy_support` rate limiting without a proxy that overwrites forwarding headers.
