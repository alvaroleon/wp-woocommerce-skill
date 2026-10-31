# Payment gateways (Checkout block)

How to build a payment method that works in the **Checkout block** on WooCommerce 11.2.1: the `WC_Payment_Gateway` contract, the mandatory block integration (PHP + JS), how the Store API calls your gateway, the React Payments settings page, money in minor units, redirect flows, webhooks, polling and refunds. Background work runs in Action Scheduler. No classic checkout: no `payment_fields()` forms, no `woocommerce_checkout_process`.

## How a payment travels

```
Checkout block (JS)                         Store API (PHP, one REST request)
-------------------                         ---------------------------------
registerPaymentMethod content
  onPaymentSetup() -> paymentMethodData ──► POST /wc/store/v1/checkout { payment_method, payment_data[] }
                                              order created/updated, set to pending, stock reserved
                                              woocommerce_store_api_checkout_order_processed
                                              order->needs_payment()? no ─► payment_complete(), redirect to order-received
                                              yes ─► session.order_awaiting_payment = order id
                                                     do_action woocommerce_rest_checkout_process_payment_with_context
                                                       prio < 999: your native handler (optional)
                                                       prio 999: Legacy bridge ─► $_POST = payment_data
                                                                               ─► validate_fields(), process_payment()
◄── payment_result { payment_status, payment_details[], redirect_url }
window.location = redirect_url (provider or order-received)
```

The shopper sees a method only if it is registered with `registerPaymentMethod`, its `AbstractPaymentMethodType::is_active()` is true, its id is in the cart response's `payment_methods` (`get_available_payment_gateways()`, i.e. `is_available()`), and `canMakePayment()` returns true. The Store API rejects a `payment_method` outside `get_available_payment_gateways()`.

## `WC_Payment_Gateway` contract

`includes/abstracts/abstract-wc-payment-gateway.php`, extends `WC_Settings_API`.

| Member | Notes |
|---|---|
| `$id` | Unique, lowercase. Must equal the block integration's `$name`. Settings in option `woocommerce_{id}_settings`. |
| `$method_title`, `$method_description` | Admin-facing (Payments page falls back to `title`/`description`). |
| `$title`, `$description` | Shopper-facing. Filters `woocommerce_gateway_title`, `woocommerce_gateway_description`. |
| `$icon` | **An image URL.** The Payments page checks `wc_is_valid_url()` and shows a default icon for an `<img>` tag. |
| `$supports` | Default `array( 'products' )`. Use `Automattic\WooCommerce\Enums\PaymentGatewayFeature`: `PRODUCTS`, `REFUNDS`, `TOKENIZATION`, `ADD_PAYMENT_METHOD`, `PRE_ORDERS`, `SUBSCRIPTIONS`, `SUBSCRIPTION_*`, `MULTIPLE_SUBSCRIPTIONS`, `DEPOSITS`, `PAY_BUTTON` (`AGENTIC_COMMERCE` deprecated 11.2.0). Checked by `supports()` (filter `woocommerce_payment_gateway_supports`). |
| `$has_fields`, `$order_button_text`, `$has_custom_place_order_button` | Classic only. In blocks: JS `content` and `placeOrderButtonLabel`. |
| `$max_amount` | `is_available()` hides the gateway above this total (0 = no limit). |
| `$view_transaction_url` | `sprintf` pattern (`%s` = transaction id) for `get_transaction_url()`. |
| `is_available()` | Since 10.7 returns early when disabled, then checks `max_amount`. Override for currency/credentials; call `parent::is_available()` first. |
| `needs_setup()` | Default `false`; shown as "needs setup" on the Payments page. |
| `process_payment( $order_id )` | Returns an array, see below. |
| `process_refund( $order_id, $amount = null, $reason = '' )` | `true` or `WP_Error`. Offered only with `REFUNDS` (`can_refund_order()`). Called by `wc_create_refund( array( 'refund_payment' => true, … ) )`, i.e. the admin refund button. |
| `get_return_url( $order )` | Order-received URL (filter `woocommerce_get_return_url`). |
| `validate_fields()` | Called by the Legacy bridge before `process_payment()`; `wc_add_notice( …, 'error' )` aborts. |

`process_payment()` results as `src/StoreApi/Legacy.php` interprets them:

| Return | Effect |
|---|---|
| `array( 'result' => 'success', 'redirect' => $url )` | `payment_status: success`; the block navigates to `redirect`. |
| `array( 'result' => 'failure', 'message' => $text )` | 400 `woocommerce_rest_payment_error` with the message (tags stripped); shopper can retry. |
| `array( 'result' => 'failure' )` | Notices from `wc_add_notice()` become the error. |
| `'pending'` / `'error'` | Accepted; anything else (or no `result`) becomes `failure`. |

The **whole returned array** is merged into `payment_result.payment_details` in the HTTP response: never put secrets in it.

Never call `WC()->cart->empty_cart()` in `process_payment()`. `wc_clear_cart_after_payment()` (`template_redirect`) empties the cart when order-received loads with a valid key, or when the session's `order_awaiting_payment` is no longer pending/failed/cancelled (filter `woocommerce_should_clear_cart_after_payment`, 9.3). A failed redirect payment keeps the cart for a retry.

## How the Store API calls your gateway

1. JS returns `meta.paymentMethodData` from `onPaymentSetup`; the block posts it as `payment_data: [ { key, value } ]`.
2. `CheckoutTrait::get_request_payment_data()` builds `array( sanitize_key( $key ) => wc_clean( $value ) )`: keys lowercased (`isSavedToken` → `issavedtoken`), booleans `'1'`/`''`. Keep values flat strings.
3. `Legacy::process_legacy_payment()` (priority 999) **replaces `$_POST` with that array**, calls `validate_fields()` and `process_payment()`, then restores `$_POST`. The Store API already checked the nonce.

### Native path (no `$_POST` swap)

Hook the same action below 999; the Legacy bridge skips a request whose `$result->status` is set. Any thrown `Exception` becomes a 400 `woocommerce_rest_checkout_process_payment_error`. Use the `StoreApi\Payments` namespace (`Blocks\Payments\PaymentResult|PaymentContext` are deprecated aliases).

```php
use Automattic\WooCommerce\StoreApi\Payments\PaymentContext;
use Automattic\WooCommerce\StoreApi\Payments\PaymentResult;

add_action( 'woocommerce_rest_checkout_process_payment_with_context', 'my_plugin_native_payment', 10, 2 );

function my_plugin_native_payment( PaymentContext $context, PaymentResult &$result ) {
	if ( 'my_plugin' !== $context->payment_method ) {
		return;
	}
	$order = $context->order;        // WC_Order, pending.
	$data  = $context->payment_data; // Sanitized flat array.
	// ... create the provider session ...
	$result->set_status( 'success' ); // success|failure|pending|error; anything else throws.
	$result->set_redirect_url( 'https://pay.example.com/session/abc' );
	$result->set_payment_details( array( 'session' => 'abc' ) ); // Public: sent to the browser.
}
```

## Checkout block integration (mandatory)

### PHP: `AbstractPaymentMethodType`

`Automattic\WooCommerce\Blocks\Payments\Integrations\AbstractPaymentMethodType`, registered on `woocommerce_blocks_payment_method_type_registration( PaymentMethodRegistry $registry )`:

| Method | Purpose |
|---|---|
| `$name` | Equals the gateway `$id`. |
| `initialize()` | Required. Load `$this->settings`. Runs on `init` priority 5. |
| `is_active()` | Default `true`. Inactive = no script, no data. |
| `get_payment_method_script_handles()` | Register (not enqueue) the script, return handles; they become dependencies of the cart/checkout scripts. A missing dependency deactivates the method ("…has been deactivated" in the log). |
| `get_payment_method_script_handles_for_admin()` | Defaults to the frontend handles. |
| `get_supported_features()` | Default `array( 'products' )`. |
| `get_payment_method_data()` | Printed into `wcSettings.paymentMethodData[ $name ]` on Cart/Checkout block pages; empty arrays dropped. |

### JS: `registerPaymentMethod`

From `@woocommerce/blocks-registry` (global `wc.wcBlocksRegistry`; externalised, never npm-installed):

| Option | Notes |
|---|---|
| `name`, `ariaLabel` | Required. `name` = gateway id. |
| `label` | React node; core uses `props.components.PaymentMethodLabel`. |
| `content`, `edit` | Required React elements (missing props throw `TypeError`). |
| `canMakePayment( { cart, cartTotals, cartNeedsShipping, shippingAddress, billingAddress, selectedShippingMethods, paymentRequirements } )` | `bool` or `Promise<bool>`, frontend only. |
| `paymentMethodId` | Defaults to `name`. |
| `supports` | `{ features, showSavedCards, showSaveOption }`; `features` defaults to `[ 'products' ]`. |
| `placeOrderButtonLabel` / `placeOrderButton` | Both set: `placeOrderButton` wins, with a warning. |
| `savedTokenComponent` | Rendered under a selected saved token. |

`content` receives `eventRegistration` (`onPaymentSetup`, `onCheckoutValidation`, `onCheckoutSuccess`, `onCheckoutFail`, `onShippingRateSuccess`, …), `emitResponse` (`responseTypes.SUCCESS`/`FAIL`/`ERROR` = `'success'`/`'failure'`/`'error'`, `noticeContexts`), `billing`, `shippingData`, `cartData`, `components`, `activePaymentMethod`, `shouldSavePayment`. `onPaymentProcessing` is deprecated → `onPaymentSetup`.

Read server data with `getPaymentMethodData( 'my_plugin', {} )` from `@woocommerce/settings` (as core gateways do). `getSetting( 'my_plugin_data', {} )` returns the same object: `getSetting()` maps `*_data` keys to `paymentMethodData`.

```js
// Express methods (wallet buttons above the checkout).
registerExpressPaymentMethod( {
	name: 'my_plugin_express',
	paymentMethodId: 'my_plugin', // Gateway that processes it.
	content: <ExpressButton />,
	edit: <ExpressButton />,
	canMakePayment: () => true,
	supports: { features: [ 'products' ], style: [ 'height', 'borderRadius' ] },
} );
```

### Calls from your gateway JS to your server

Shopper-facing, so Store API (premise 5): a route under **`wc/store/v1`**, never `admin-ajax.php`, `wc-ajax` or your own namespace. Any route starting with `/wc/store/` gets frontend includes, the Store API session handler (cookie or `Cart-Token`), the `Nonce`/`Cart-Token` CORS headers and rate limiting. **Not** automatic for a route you register: the nonce check (core does it inside its own route classes) and loading the cart.

```php
add_action( 'rest_api_init', 'my_plugin_register_store_routes' );

function my_plugin_register_store_routes() {
	register_rest_route(
		'wc/store/v1',
		'/my-plugin/session', // POST /wp-json/wc/store/v1/my-plugin/session
		array(
			'methods'             => 'POST',
			'permission_callback' => static function ( WP_REST_Request $request ) {
				return (bool) wp_verify_nonce( (string) $request->get_header( 'Nonce' ), 'wc_store_api' );
			},
			'callback'            => 'my_plugin_create_intent',
		)
	);
}

function my_plugin_create_intent() {
	if ( null === WC()->cart ) {
		wc_load_cart(); // Not loaded automatically for REST requests.
	}
	WC()->cart->calculate_totals();
	$currency = get_woocommerce_currency();
	$intent   = my_plugin_api( 'POST', '/intents', array( 'amount' => my_plugin_to_minor_units( (string) WC()->cart->get_total( 'edit' ), $currency ), 'currency' => $currency ) );
	if ( is_wp_error( $intent ) || empty( $intent['client_secret'] ) ) {
		return new WP_Error( 'my_plugin_intent', __( 'Payment is unavailable right now.', 'my-plugin' ), array( 'status' => 502 ) );
	}
	return rest_ensure_response( array( 'client_secret' => (string) $intent['client_secret'] ) );
}
```

```js
import apiFetch from '@wordpress/api-fetch';
// On cart/checkout pages core's wc-blocks-middleware adds the Nonce header to every non-GET
// apiFetch call whose path contains wc/store/v1/, custom routes included.
const { client_secret: clientSecret } = await apiFetch( { path: '/wc/store/v1/my-plugin/session', method: 'POST' } );
```

**Provider webhooks are not shopper-facing**: server-to-server, no session/cart/nonce, authenticated by signature. They stay in your own namespace (`my-plugin/v1/webhook`, below), not under `wc/store/v1` where Store API rate limiting (when enabled) would count the provider's IP.

## Payments settings page (React)

Force-enabled (`woocommerce_feature_reactify-classic-payments-settings_enabled = yes`, update 985). Your `WC_Settings_API` form still renders at the gateway's settings URL. The page reads optional, duck-typed methods (errors logged under `settings-payments`); there are no other extension hooks:

| Method on your gateway | Used for |
|---|---|
| `get_settings_url()` | "Manage" link. Default `admin.php?page=wc-settings&tab=checkout&section={id}`. |
| `needs_setup()` | "Needs setup" state. |
| `is_account_connected()` / `is_connected()` | Connection state (default **true**). |
| `is_test_mode()` / `is_in_test_mode()`, else props `testmode`/`test_mode`, else options `test_mode`, `testmode`, `mode` (`test`/`sandbox`/`dev` vs `live`/`production`/`prod`) | Test-mode badge. |
| `is_dev_mode()` / `is_in_dev_mode()` | Dev-mode badge. |
| `is_onboarding_supported( $country )`, `is_onboarding_started()`, `is_onboarding_completed()`, `get_connection_url( $return_url )`, `get_recommended_payment_methods( $country )`, `get_provider_links( $country )` | Onboarding (optional). |

Gateway order is stored in `woocommerce_gateway_order`.

## Money in minor units

- `$order->get_total()` is a decimal **string** in store decimals. Never `* 100`: float math, and 100× overcharge for zero-decimal currencies (CLP has `num_decimals` 0 in `i18n/locale-info.php`).
- Convert with the **provider's** ISO 4217 exponent for the order currency, not `wc_get_price_decimals()` (a filterable display setting).
- Store API prices (`/cart`, rates) are integer strings in minor units of the **store** decimals, with `currency_minor_unit` in the response.
- Compare the provider's amount **and** currency with the order before `payment_complete()`.

Transbank Webpay Plus (CLP only) sends `(int) number_format( $order->get_total(), 0, ',', '' )`. The helpers below do it for any currency without floats.

## Complete example: redirect gateway with webhooks and polling

```
my-plugin/
├── my-plugin.php                          header + compat (SKILL.md boilerplate), money, API, webhook, jobs
├── includes/class-my-plugin-gateway.php
├── includes/class-my-plugin-blocks.php
├── src/index.js  →  build/index.js + build/index.asset.php   (npm run build)
├── assets/icon.svg, package.json, webpack.config.js
```

`my-plugin.php` (after the SKILL.md plugin header and `declare_compatibility` calls; defines `MY_PLUGIN_FILE`):

```php
use Automattic\WooCommerce\Enums\OrderStatus;

define( 'MY_PLUGIN_FILE', __FILE__ );

add_filter( 'woocommerce_payment_gateways', 'my_plugin_add_gateway' );
add_action( 'woocommerce_blocks_payment_method_type_registration', 'my_plugin_register_block_method' );
add_action( 'rest_api_init', 'my_plugin_register_webhook_route' );
add_action( 'my_plugin_process_event', 'my_plugin_process_event', 10, 3 );
add_action( 'my_plugin_poll_payment', 'my_plugin_poll_payment', 10, 2 );

function my_plugin_add_gateway( array $gateways ): array {
	require_once __DIR__ . '/includes/class-my-plugin-gateway.php';
	$gateways[] = 'My_Plugin_Gateway';
	return $gateways;
}

function my_plugin_register_block_method( $registry ) {
	require_once __DIR__ . '/includes/class-my-plugin-blocks.php';
	$registry->register( new My_Plugin_Blocks() );
}

/** ISO 4217 exponent. Check your provider's own table. */
function my_plugin_currency_exponent( string $currency ): int {
	$currency = strtoupper( $currency );
	if ( in_array( $currency, array( 'BIF', 'CLP', 'DJF', 'GNF', 'ISK', 'JPY', 'KMF', 'KRW', 'PYG', 'RWF', 'UGX', 'VND', 'VUV', 'XAF', 'XOF', 'XPF' ), true ) ) {
		return 0;
	}
	return in_array( $currency, array( 'BHD', 'IQD', 'JOD', 'KWD', 'LYD', 'OMR', 'TND' ), true ) ? 3 : 2;
}

/** '15990' CLP -> 15990; '12.50' USD -> 1250. No float multiplication. */
function my_plugin_to_minor_units( string $amount, string $currency ): int {
	return (int) str_replace( '.', '', wc_format_decimal( $amount, my_plugin_currency_exponent( $currency ) ) );
}

/** 1250 USD -> '12.50'; 15990 CLP -> '15990'. */
function my_plugin_from_minor_units( int $minor, string $currency ): string {
	$exp = my_plugin_currency_exponent( $currency );
	if ( 0 === $exp ) {
		return (string) $minor;
	}
	$digits = str_pad( (string) abs( $minor ), $exp + 1, '0', STR_PAD_LEFT );
	return ( $minor < 0 ? '-' : '' ) . substr( $digits, 0, -$exp ) . '.' . substr( $digits, -$exp );
}

/** Minimal provider client: short timeout, returns array or WP_Error. */
function my_plugin_api( string $method, string $path, array $body = array() ) {
	$settings = get_option( 'woocommerce_my_plugin_settings', array() );
	$base     = 'yes' === ( $settings['test_mode'] ?? 'yes' ) ? 'https://sandbox.example-provider.test/v1' : 'https://api.example-provider.test/v1';
	$response = wp_remote_request(
		$base . $path,
		array(
			'method'  => $method,
			'timeout' => 10,
			'headers' => array( 'Authorization' => 'Bearer ' . ( $settings['api_key'] ?? '' ), 'Content-Type' => 'application/json' ),
			'body'    => $body ? wp_json_encode( $body ) : null,
		)
	);
	$code = (int) wp_remote_retrieve_response_code( $response );
	$data = is_wp_error( $response ) ? null : json_decode( wp_remote_retrieve_body( $response ), true );
	return ( $code >= 200 && $code < 300 && is_array( $data ) ) ? $data : new WP_Error( 'my_plugin_api', sprintf( 'Provider HTTP %d', $code ) );
}

/**
 * Applies the provider's state to the order. Idempotent: safe from the return URL, the
 * webhook job and the poll job, in any order, any number of times.
 *
 * @param array $payment { id, status: paid|failed|pending, amount (minor), currency, refunds: [ { id, amount } ] }.
 */
function my_plugin_apply_payment_state( WC_Order $order, array $payment ): void {
	if ( (string) $order->get_transaction_id() !== (string) $payment['id'] ) {
		return;
	}
	if ( 'paid' === $payment['status'] && ! $order->is_paid() ) {
		$expected = my_plugin_to_minor_units( (string) $order->get_total(), $order->get_currency() );
		if ( $expected !== (int) $payment['amount'] || strtoupper( $order->get_currency() ) !== strtoupper( (string) $payment['currency'] ) ) {
			$order->update_status( OrderStatus::ON_HOLD, __( 'Provider amount or currency does not match the order.', 'my-plugin' ) );
			return;
		}
		$order->payment_complete( (string) $payment['id'] ); // Adds its own "Payment via" note.
	} elseif ( 'failed' === $payment['status'] && $order->has_status( OrderStatus::PENDING ) ) {
		$order->update_status( OrderStatus::FAILED, __( 'Payment declined by the provider.', 'my-plugin' ) );
	}

	// Refunds made in the provider dashboard (refunds made in WooCommerce were recorded by process_refund()).
	$known = $order->get_meta( '_my_plugin_refund_ids' );
	$known = is_array( $known ) ? $known : array();
	foreach ( (array) ( $payment['refunds'] ?? array() ) as $refund ) {
		if ( in_array( (string) $refund['id'], $known, true ) ) {
			continue;
		}
		$created = wc_create_refund(
			array(
				'order_id'       => $order->get_id(),
				'amount'         => my_plugin_from_minor_units( (int) $refund['amount'], $order->get_currency() ),
				'reason'         => __( 'Refunded in the provider dashboard.', 'my-plugin' ),
				'refund_payment' => false, // Money already moved: do not call process_refund().
			)
		);
		if ( ! is_wp_error( $created ) ) {
			$known[] = (string) $refund['id'];
		}
	}
	$order = wc_get_order( $order->get_id() ); // Refunds may have changed the status.
	$order->update_meta_data( '_my_plugin_refund_ids', $known );
	$order->save();
}

// Server-to-server webhook: own namespace (not wc/store/v1). Verify, enqueue, answer fast.
function my_plugin_register_webhook_route() {
	register_rest_route(
		'my-plugin/v1',
		'/webhook',
		array(
			'methods'             => 'POST',
			'permission_callback' => '__return_true', // Authenticated by the HMAC below.
			'callback'            => 'my_plugin_handle_webhook',
		)
	);
}

function my_plugin_handle_webhook( WP_REST_Request $request ): WP_REST_Response {
	$settings  = get_option( 'woocommerce_my_plugin_settings', array() );
	$secret    = (string) ( $settings['webhook_secret'] ?? '' );
	$raw       = $request->get_body(); // Raw bytes: never re-encode before hashing.
	$signature = (string) $request->get_header( 'x-provider-signature' );
	if ( '' === $secret || '' === $signature || ! hash_equals( hash_hmac( 'sha256', $raw, $secret ), $signature ) ) {
		return new WP_REST_Response( array( 'ok' => false ), 401 );
	}
	$event      = json_decode( $raw, true );
	$event_id   = sanitize_text_field( (string) ( $event['id'] ?? '' ) );
	$payment_id = sanitize_text_field( (string) ( $event['data']['payment_id'] ?? '' ) );
	if ( '' === $event_id || '' === $payment_id ) {
		return new WP_REST_Response( array( 'ok' => false ), 400 );
	}
	// Unique: no second pending/running action with the same hook + group + args (redelivered event).
	as_enqueue_async_action( 'my_plugin_process_event', array( $payment_id, $event_id, 1 ), 'my-plugin', true );
	return new WP_REST_Response( array( 'ok' => true ), 200 );
}

// The payload is only a signal: re-read the payment from the provider (source of truth).
function my_plugin_process_event( $payment_id, $event_id, $attempt ) {
	$orders = wc_get_orders( array( 'transaction_id' => $payment_id, 'limit' => 1 ) ); // HPOS column.
	if ( ! $orders ) {
		return;
	}
	$payment = my_plugin_api( 'GET', '/payments/' . rawurlencode( $payment_id ) );
	if ( is_wp_error( $payment ) ) {
		if ( $attempt < 6 ) { // Failed actions are never retried by Action Scheduler: back off yourself.
			as_schedule_single_action( time() + 60 * ( 2 ** $attempt ), 'my_plugin_process_event', array( $payment_id, $event_id, $attempt + 1 ), 'my-plugin', true );
		}
		return;
	}
	my_plugin_apply_payment_state( $orders[0], $payment );
}

// Safety net for lost webhooks, first scheduled by process_payment().
function my_plugin_poll_payment( $order_id, $attempt ) {
	$order = wc_get_order( $order_id );
	if ( ! $order || ! $order->has_status( OrderStatus::PENDING ) ) {
		return;
	}
	$payment = my_plugin_api( 'GET', '/payments/' . rawurlencode( $order->get_transaction_id() ) );
	if ( ! is_wp_error( $payment ) ) {
		my_plugin_apply_payment_state( $order, $payment );
		$order = wc_get_order( $order_id );
	}
	if ( $order->has_status( OrderStatus::PENDING ) && $attempt < 8 ) {
		$delay = min( HOUR_IN_SECONDS, 2 * MINUTE_IN_SECONDS * ( 2 ** $attempt ) );
		as_schedule_single_action( time() + $delay, 'my_plugin_poll_payment', array( $order_id, $attempt + 1 ), 'my-plugin', true );
	}
}
```

`includes/class-my-plugin-gateway.php`:

```php
<?php
defined( 'ABSPATH' ) || exit;

use Automattic\WooCommerce\Enums\OrderStatus;
use Automattic\WooCommerce\Enums\PaymentGatewayFeature;

class My_Plugin_Gateway extends WC_Payment_Gateway {

	public function __construct() {
		$this->id                 = 'my_plugin';
		$this->method_title       = __( 'My Plugin Pay', 'my-plugin' );
		$this->method_description = __( 'Sends the shopper to the provider to pay.', 'my-plugin' );
		$this->icon               = plugins_url( 'assets/icon.svg', MY_PLUGIN_FILE ); // URL, not <img>.
		$this->supports           = array( PaymentGatewayFeature::PRODUCTS, PaymentGatewayFeature::REFUNDS );
		$this->init_form_fields();
		$this->init_settings();
		$this->title       = $this->get_option( 'title' );
		$this->description = $this->get_option( 'description' );

		add_action( 'woocommerce_update_options_payment_gateways_' . $this->id, array( $this, 'process_admin_options' ) );
		// Customer return: /wc-api/my_plugin_return/. The dispatcher loads gateways first and sends no-cache headers.
		add_action( 'woocommerce_api_my_plugin_return', array( $this, 'handle_return' ) );
	}

	public function init_form_fields() {
		$this->form_fields = array(
			'enabled'        => array( 'title' => __( 'Enable', 'my-plugin' ), 'type' => 'checkbox', 'default' => 'no', 'label' => __( 'Enable My Plugin Pay', 'my-plugin' ) ),
			'title'          => array( 'title' => __( 'Title', 'my-plugin' ), 'type' => 'text', 'default' => __( 'Pay online', 'my-plugin' ) ),
			'description'    => array( 'title' => __( 'Description', 'my-plugin' ), 'type' => 'textarea', 'default' => __( 'You will be redirected to complete the payment.', 'my-plugin' ) ),
			'test_mode'      => array( 'title' => __( 'Test mode', 'my-plugin' ), 'type' => 'checkbox', 'default' => 'yes', 'label' => __( 'Use the sandbox', 'my-plugin' ) ), // Read by the Payments page.
			'api_key'        => array( 'title' => __( 'API key', 'my-plugin' ), 'type' => 'password', 'default' => '' ),
			'webhook_secret' => array( 'title' => __( 'Webhook secret', 'my-plugin' ), 'type' => 'password', 'default' => '', 'description' => esc_html( rest_url( 'my-plugin/v1/webhook' ) ) ),
		);
	}

	public function needs_setup() {
		return '' === $this->get_option( 'api_key' );
	}

	public function is_available() {
		return parent::is_available() && ! $this->needs_setup() && in_array( get_woocommerce_currency(), array( 'CLP', 'USD' ), true );
	}

	public function process_payment( $order_id ) {
		$order = wc_get_order( $order_id );
		if ( ! $order ) {
			return array( 'result' => 'failure', 'message' => __( 'Order not found.', 'my-plugin' ) );
		}
		// $_POST is the Store API payment_data here (sanitized; nonce already checked).
		$installments = isset( $_POST['my_plugin_installments'] ) && 'yes' === $_POST['my_plugin_installments']; // phpcs:ignore WordPress.Security.NonceVerification.Missing
		$return_url   = add_query_arg( array( 'order_id' => $order->get_id(), 'key' => $order->get_order_key() ), WC()->api_request_url( 'my_plugin_return' ) );

		$session = my_plugin_api(
			'POST',
			'/sessions',
			array(
				'amount'       => my_plugin_to_minor_units( (string) $order->get_total(), $order->get_currency() ),
				'currency'     => $order->get_currency(),
				'reference'    => (string) $order->get_id(),
				'installments' => $installments,
				'return_url'   => $return_url,
			)
		);
		if ( is_wp_error( $session ) || empty( $session['id'] ) || empty( $session['redirect_url'] ) ) {
			wc_get_logger()->error( 'Session creation failed for order ' . $order->get_id(), array( 'source' => 'my-plugin' ) );
			return array( 'result' => 'failure', 'message' => __( 'We could not start the payment. Please try again.', 'my-plugin' ) );
		}

		$order->set_transaction_id( sanitize_text_field( (string) $session['id'] ) ); // Lookup key for webhooks.
		$order->save();
		as_schedule_single_action( time() + 2 * MINUTE_IN_SECONDS, 'my_plugin_poll_payment', array( $order->get_id(), 1 ), 'my-plugin', true );

		return array( 'result' => 'success', 'redirect' => esc_url_raw( $session['redirect_url'] ) );
	}

	/** Never trust the provider's query params: check the order key, then ask the provider. */
	public function handle_return() {
		// phpcs:disable WordPress.Security.NonceVerification.Recommended -- The order key is the credential.
		$order_id = isset( $_GET['order_id'] ) ? absint( $_GET['order_id'] ) : 0;
		$key      = isset( $_GET['key'] ) ? sanitize_text_field( wp_unslash( $_GET['key'] ) ) : '';
		// phpcs:enable
		$order = $order_id ? wc_get_order( $order_id ) : false;
		if ( ! $order || ! hash_equals( $order->get_order_key(), $key ) || $this->id !== $order->get_payment_method() ) {
			wp_safe_redirect( wc_get_cart_url() );
			exit;
		}
		if ( ! $order->is_paid() ) {
			$payment = my_plugin_api( 'GET', '/payments/' . rawurlencode( $order->get_transaction_id() ) );
			if ( ! is_wp_error( $payment ) ) {
				my_plugin_apply_payment_state( $order, $payment );
				$order = wc_get_order( $order_id );
			}
		}
		$failed = $order->has_status( array( OrderStatus::FAILED, OrderStatus::CANCELLED ) );
		// Failed: back to checkout with the cart intact. Otherwise order-received, where core clears the cart.
		wp_safe_redirect( $failed ? wc_get_checkout_url() : $this->get_return_url( $order ) );
		exit;
	}

	public function process_refund( $order_id, $amount = null, $reason = '' ) {
		$order = wc_get_order( $order_id );
		if ( ! $order || '' === (string) $order->get_transaction_id() || null === $amount || 0 >= (float) $amount ) {
			return new WP_Error( 'my_plugin_refund', __( 'This order cannot be refunded online.', 'my-plugin' ) );
		}
		$path   = '/payments/' . rawurlencode( $order->get_transaction_id() ) . '/refunds';
		$refund = my_plugin_api( 'POST', $path, array( 'amount' => my_plugin_to_minor_units( (string) $amount, $order->get_currency() ), 'reason' => $reason ) );
		if ( is_wp_error( $refund ) || empty( $refund['id'] ) ) {
			return new WP_Error( 'my_plugin_refund', __( 'The provider rejected the refund.', 'my-plugin' ) );
		}
		$known   = $order->get_meta( '_my_plugin_refund_ids' );
		$known   = is_array( $known ) ? $known : array();
		$known[] = (string) $refund['id']; // So the webhook does not create it again.
		$order->update_meta_data( '_my_plugin_refund_ids', $known );
		$order->save();
		/* translators: %s: provider refund id */
		$order->add_order_note( sprintf( __( 'Refund %s accepted by the provider.', 'my-plugin' ), $refund['id'] ) );
		return true;
	}
}
```

`includes/class-my-plugin-blocks.php`:

```php
<?php
defined( 'ABSPATH' ) || exit;

use Automattic\WooCommerce\Blocks\Payments\Integrations\AbstractPaymentMethodType;

final class My_Plugin_Blocks extends AbstractPaymentMethodType {

	protected $name = 'my_plugin'; // === gateway id.

	public function initialize() {
		$this->settings = get_option( 'woocommerce_my_plugin_settings', array() );
	}

	public function is_active() {
		return filter_var( $this->get_setting( 'enabled', false ), FILTER_VALIDATE_BOOLEAN );
	}

	public function get_payment_method_script_handles() {
		$asset = include plugin_dir_path( MY_PLUGIN_FILE ) . 'build/index.asset.php';
		wp_register_script( 'my-plugin-blocks', plugins_url( 'build/index.js', MY_PLUGIN_FILE ), $asset['dependencies'], $asset['version'], true );
		wp_set_script_translations( 'my-plugin-blocks', 'my-plugin', plugin_dir_path( MY_PLUGIN_FILE ) . 'languages' );
		return array( 'my-plugin-blocks' );
	}

	public function get_supported_features() {
		$gateways = WC()->payment_gateways()->payment_gateways();
		$gateway  = $gateways[ $this->name ] ?? null;
		return $gateway ? array_values( array_filter( $gateway->supports, array( $gateway, 'supports' ) ) ) : array( 'products' );
	}

	/** Static config only: printed into wcSettings on the cart and checkout pages. */
	public function get_payment_method_data() {
		return array(
			'title'       => $this->get_setting( 'title' ),
			'description' => $this->get_setting( 'description' ),
			'icon'        => plugins_url( 'assets/icon.svg', MY_PLUGIN_FILE ),
			'supports'    => $this->get_supported_features(),
		);
	}
}
```

`src/index.js`:

```js
import { registerPaymentMethod } from '@woocommerce/blocks-registry';
import { getPaymentMethodData } from '@woocommerce/settings';
import { decodeEntities } from '@wordpress/html-entities';
import { useEffect, useState } from '@wordpress/element';
import { __ } from '@wordpress/i18n';

const settings = getPaymentMethodData( 'my_plugin', {} );
const title = decodeEntities( settings.title || '' ) || __( 'Pay online', 'my-plugin' );

const Content = ( { eventRegistration, emitResponse } ) => {
	const { onPaymentSetup } = eventRegistration;
	const [ installments, setInstallments ] = useState( false );

	// Arrives in process_payment() as $_POST['my_plugin_installments'].
	useEffect(
		() =>
			onPaymentSetup( () => ( {
				type: emitResponse.responseTypes.SUCCESS,
				meta: { paymentMethodData: { my_plugin_installments: installments ? 'yes' : 'no' } },
			} ) ),
		[ onPaymentSetup, emitResponse.responseTypes.SUCCESS, installments ]
	);

	return (
		<>
			<p>{ decodeEntities( settings.description || '' ) }</p>
			<label>
				<input type="checkbox" checked={ installments } onChange={ ( e ) => setInstallments( e.target.checked ) } />{ ' ' }
				{ __( 'Pay in installments', 'my-plugin' ) }
			</label>
		</>
	);
};

const Label = ( { components } ) => <components.PaymentMethodLabel text={ title } />;

registerPaymentMethod( {
	name: 'my_plugin',
	label: <Label />,
	ariaLabel: title,
	content: <Content />,
	edit: <Content />,
	canMakePayment: ( { cartTotals } ) => [ 'CLP', 'USD' ].includes( cartTotals.currency_code ),
	supports: { features: settings.supports ?? [ 'products' ] },
} );
```

Build with `@wordpress/scripts`, swapping in the WooCommerce extraction plugin so `@woocommerce/*` imports become `wc.*` globals and `index.asset.php` lists `wc-blocks-registry` / `wc-settings`:

```js
// webpack.config.js. package.json: "build": "wp-scripts build"; devDependencies
// @wordpress/scripts and @woocommerce/dependency-extraction-webpack-plugin.
const defaultConfig = require( '@wordpress/scripts/config/webpack.config' );
const WooCommerceDependencyExtractionWebpackPlugin = require( '@woocommerce/dependency-extraction-webpack-plugin' );

module.exports = {
	...defaultConfig,
	plugins: [
		...defaultConfig.plugins.filter( ( plugin ) => plugin.constructor.name !== 'DependencyExtractionWebpackPlugin' ),
		new WooCommerceDependencyExtractionWebpackPlugin(),
	],
};
```

Why the webhook is shaped this way:

- **Own namespace, raw body, `hash_equals`.** Not shopper-facing, so not `wc/store/v1`. HMAC over `$request->get_body()`, constant-time comparison, 401 on mismatch.
- **Enqueue, do not process.** Action Scheduler uniqueness is hook + group + args among pending/running actions: a redelivered event is not queued twice; different events for one payment get separate jobs.
- **Idempotent apply.** The job re-fetches the payment; `is_paid()` guards `payment_complete()` (a second call only fires `woocommerce_payment_complete_order_status_{status}`), refunds are deduplicated by provider refund id. A late redelivery after completion is harmless.
- **Amount + currency check** before `payment_complete()`; mismatch → on-hold with a note.
- **`wc_create_refund()`** (refund object, line totals, refund emails), never `update_status( 'refunded' )`.
- **Retries are yours**: failed actions are not retried. See `action-scheduler.md`.

## Saved tokens in blocks

Verified behaviour only:

- The Checkout block hydrates the logged-in customer's saved methods as `customerPaymentMethods` (checkout is never cached).
- JS `supports.showSavedCards` / `showSaveOption` enable the saved list and the "save" checkbox; `savedTokenComponent` renders under a selected token.
- With a saved token, `payment_data` holds `token`, `payment_method`, `wc-{gateway_id}-payment-token` (the `WC_Payment_Token` id) and `issavedtoken`. With the save option shown it adds `wc-{gateway_id}-new-payment-method` (`'1'` or `''`).
- In `process_payment()`: `WC_Payment_Tokens::get( absint( $_POST[ 'wc-' . $this->id . '-payment-token' ] ) )`, then require `get_user_id() === $order->get_customer_id()` and `get_gateway_id() === $this->id`.
- `WC_Payment_Token_CC` (`set_card_type`, `set_last4`, `set_expiry_month`, `set_expiry_year`), `WC_Payment_Token_ECheck`. `WC_Payment_Gateway_CC` is the classic card form: unused in blocks.

## Full-page cache notes

- `get_payment_method_data()` is printed into `wcSettings` on the Cart and Checkout pages: **static config only** (title, description, icon URL, public keys, features). No nonces, totals, customer data or session ids.
- Cart and checkout pages also carry hydrated Store API data and saved methods: they must stay out of the HTML cache (WooCommerce sends no-cache headers; edge rules must agree).
- Dynamic data is fetched by JS at runtime: totals from the `wc/store/cart` data store, intents from your `wc/store/v1/my-plugin/…` route (see `cache-and-store-api.md`).
- An express button on a cached product or home page renders client-side and reads amounts from the Store API when clicked, never from the HTML.

## Common mistakes

- A `WC_Payment_Gateway` without `AbstractPaymentMethodType` + `registerPaymentMethod`: invisible in the Checkout block.
- Block integration `$name` different from the gateway `$id`.
- Bundling `@woocommerce/blocks-registry`, or building without `@woocommerce/dependency-extraction-webpack-plugin`: wrong dependencies, method deactivated.
- `payment_fields()` forms or `woocommerce_checkout_process`: neither runs in blocks. Collect in `content`, send via `onPaymentSetup`, validate in `validate_fields()` or the native handler.
- `onPaymentProcessing` (deprecated) instead of `onPaymentSetup`.
- Nested arrays or camelCase keys in `paymentMethodData` (`wc_clean` + `sanitize_key`).
- Secrets in the `process_payment()` return array: echoed to the browser as `payment_details`.
- `$order->get_total() * 100`, or `wc_get_price_decimals()` as the provider exponent.
- `WC()->cart->empty_cart()` in `process_payment()`.
- Marking the order paid from the return URL's query string: always verify with the provider.
- Gateway JS calling `admin-ajax.php`, `wc-ajax` or a custom namespace for shopper actions: use `wc/store/v1/my-plugin/…` and verify the `wc_store_api` nonce there (core does not for routes you register).
- Registering the provider webhook under `wc/store/v1`: it is server-to-server.
- Webhook on `woocommerce_api_*` processing inline and ending with `wp_die()`; no signature check; trusting `order_id` from the payload; `update_status( 'refunded' )`.
- `wp_schedule_single_event()` for polling instead of `as_schedule_single_action( …, true )`.
- `$icon` as an `<img>` tag.
- `(array) $order->get_meta( 'missing' )` is `array( '' )`, not an empty array.
