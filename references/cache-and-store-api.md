# Full-page cache and the Store API

How a WooCommerce 11.x shop serves catalogue pages from a page/edge cache that is identical for every visitor, while the cart, session and nonces reach the browser at runtime from the Store API. Everything here is verified against the 11.2.1 source unless marked **untested**. Read this before writing any code that touches the cart from the browser, renders cart-related markup, or extends the Store API.

## The model end to end

```
 Browser                              Page / edge cache                WordPress + WooCommerce
 ───────                              ─────────────────                ───────────────────────
 GET /product/x  ───────────────────▶ HIT → shared HTML ──── MISS ───▶ render with the EMPTY-cart seed
                                      (stores it only if no             (guest with no session: no WC cookie,
                                       Set-Cookie / no-store)            no nocache headers → cacheable)
 HTML loads the iAPI Mini-Cart module
   └─ module init: GET /wp-json/wc/store/v1/cart ───── bypass ───────▶ Store API reads the session cookie
      (cache: 'no-store', same-origin cookies)                          (if any) and answers the real cart +
   ◀─────────────────────────────────────────────────────────────────── Nonce, Nonce-Timestamp, Cart-Hash,
   state.cart = real cart · state.nonce = Nonce header                   Cache-Control: no-store
 "Add to cart" → POST /wc/store/v1/batch  (Nonce header) ────────────▶ adds the item; the first item sets
   ◀── Set-Cookie: wp_woocommerce_session_* (HttpOnly),                  the session + cart cookies
       woocommerce_items_in_cart=1, woocommerce_cart_hash=…
 document.body dispatches wc-blocks_added_to_cart → Mini-Cart refreshCartItems
```

Why a cached page is always correct: server-side, `BlocksSharedState::load_cart_state()` puts the `/wc/store/v1/cart` body into `wp_interactivity_state( 'woocommerce', [ 'cart' => … ] )` and calls `nocache_headers()` **only when the cart is non-empty**. So the only HTML a cache can store carries the empty-cart seed, which is right for anyone until the client refresh lands. The shared iAPI cart store (`@woocommerce/stores/woocommerce/cart`) calls `refreshCartItems()` at module init: `GET /wc/store/v1/cart` with `cache: 'no-store'`, takes `Nonce` from the response header, and every mutation (a batch `POST /wc/store/v1/batch` with that `Nonce`) waits for that first GET. The cost: one uncached Store API request per page view on pages that carry the Mini-Cart or Product Button.

## Session and cookies

| Cookie | Set by | HttpOnly | Meaning |
|---|---|---|---|
| `wp_woocommerce_session_<COOKIEHASH>` | `WC_Session_Handler::set_customer_session_cookie()`, hooked to `woocommerce_set_cart_cookies` (cart becomes non-empty) and to `wp` for `order-pay` | yes | The cart session. The only identity a guest has. |
| `woocommerce_items_in_cart` | `WC_Cart_Session::set_cart_cookies()` via `maybe_set_cart_cookies()` on `woocommerce_add_to_cart`, `wp` (99) and `shutdown` (0) | no | "This browser probably has items." A hint, readable by JS. |
| `woocommerce_cart_hash` | same | no | Hash of the cart contents. |
| `wordpress_logged_in_*` | WordPress | yes | Logged-in user → never serve cached HTML. |

- An anonymous visitor with an empty cart has **no** WooCommerce cookie. That is what makes the first page cacheable.
- A Store API add-to-cart sets the cart and session cookies in its own response (`woocommerce_add_to_cart` fires before output). Removing the last item through the Store API unsets them only if headers are still unsent at `shutdown`, or on the next uncached page load (`wp`, priority 99), so a stale `woocommerce_items_in_cart=1` can survive on cached pages. Treat the cookie as "maybe items"; its absence as "empty as of the last server response".
- Same-origin `fetch` sends the session cookie with `credentials: 'same-origin'` (the default for `fetch`; state it explicitly anyway).
- The React cart store (`wc/store/cart`, used by the Cart/Checkout blocks and legacy React blocks) gates on these cookies: it only trusts `localStorage.storeApiCartData` when `woocommerce_items_in_cart` exists **and** `localStorage.storeApiCartHash` equals the `woocommerce_cart_hash` cookie; with no items cookie it skips the fetch and assumes empty. Each response's `Cart-Hash` header updates `storeApiCartHash`. The iAPI store does not use this gate; it always fetches.

## The nonce is a runtime value

- Every Store API cart/checkout response sends `Nonce`, `Nonce-Timestamp`, `User-ID` and `Cache-Control: no-store`, plus `Cart-Token` and `Cart-Hash` when cart updates are enabled (`AbstractCartRoute::add_response_headers()`).
- Write requests (`POST`/`PUT`/`PATCH`/`DELETE`) need header `Nonce: <wc_store_api nonce>` unless a valid `Cart-Token` header is sent. Missing → `401 woocommerce_rest_missing_nonce`; invalid → `403 woocommerce_rest_invalid_nonce`. `GET` needs nothing.
- For guests the nonce is computed for user 0: WooCommerce only remaps `nonce_user_logged_out` to the session customer id for actions starting with `woocommerce`, and the action here is `wc_store_api`. So a guest nonce is shared by all guests and expires with normal nonce ticks; a login changes it.
- `wc-blocks-middleware` (loaded with `wc-blocks-data-store`, i.e. by React blocks and by `wc-blocks-checkout`) keeps the nonce in `localStorage.storeApiNonce` as JSON `{ "nonce": "…", "timestamp": 1730000000 }`, updates it from each response with the newer timestamp, and attaches `Nonce` to non-GET `wc/store/v1/*` calls made through `wp.apiFetch`, including batch sub-requests.
- **11.2.1 seed bug:** PHP prints `wcBlocksMiddlewareConfig = { storeApiNonce, wcStoreApiNonceTimestamp }` but the JS reads `storeApiNonceTimestamp`. With the timestamp undefined, the inline seed replaces the stored nonce whenever they differ, so on a cached page the stale seed beats a fresher `localStorage.storeApiNonce` until the next Store API response corrects it. Practical rule for custom code on cached pages: **obtain a nonce from a `GET /wc/store/v1/cart` response (or retry once after a 403 `woocommerce_rest_invalid_nonce`) before the first write.** Effects beyond that are untested.
- The iAPI cart store never reads the HTML seed or `localStorage`; it keeps its own `state.nonce` from its GET.
- Testing only: `add_filter( 'woocommerce_store_api_disable_nonce_check', '__return_true' );`. Never in production.

## Cart-Token: what it is and why you must not store one

`Cart-Token` is a signed token that identifies a cart without cookies, for headless or cross-origin clients. A request carrying a valid `Cart-Token` header switches to the token-keyed session handler, skips the nonce requirement, and skips cart-cookie updates. Every Store API response hands you one, which is why it is tempting to persist it. Don't: the Cart and Checkout blocks keep using the cookie session, so a token you replay from `localStorage` is a second, diverging cart (items added on the product page missing at `/checkout`). It expires with the WooCommerce session (`wc_session_expiration`, 2 days by default). On a same-origin storefront: cookie + runtime nonce, never a token.

## What to cache and what to bypass

| Request | Edge / page cache |
|---|---|
| Home, shop, category, tag, product, CMS pages for guests | Cache. HTML carries no per-user data. |
| Cart, checkout, my-account (and endpoints) | Bypass. WooCommerce also sends no-cache headers there (`WC_Cache_Helper::prevent_caching()` on `wp_headers`). |
| `/wp-json/*` and `?rest_route=` (includes the whole Store API) | Bypass all of it. Cart routes send `Cache-Control: no-store`. |
| `?wc-ajax=*`, `?add-to-cart=*`, `wp-admin`, `wp-login.php`, previews | Bypass. |
| Requests with a `wordpress_logged_in_*` cookie | Bypass. |
| Any response with `Set-Cookie`, `no-store` or `private` | Never store. Check that your edge's TTL override does not ignore these. |
| Requests with cart cookies (`woocommerce_items_in_cart`, `wp_woocommerce_session_*`) | No bypass needed: a HIT serves the empty-cart seed and the client refresh fills the real cart; a MISS for that visitor renders their cart and sends `nocache_headers()`, so it is not stored. |

Keep HTML TTL well under the nonce lifetime and purge on deploy (see below), and don't let the edge set a long browser TTL on HTML: a purge clears the edge, not the visitor's browser copy.

## Which blocks put cart data where

| Block / asset | Cart data in HTML? | Notes |
|---|---|---|
| Mini-Cart (iAPI; the only implementation since 11.1) | Badge, subtotal, and the full cart body in iAPI state; non-empty cart → `nocache_headers()` | Refreshes on load and on `wc-blocks_added_to_cart` / `wc-blocks_removed_from_cart` (listened on `document`). Opens its drawer on add when `addToCartBehaviour` is `open_drawer`. Returns `''` for logged-out visitors in store-pages-only coming-soon mode. Enqueues every script registered on `woocommerce_blocks_mini-cart_block_registration`. |
| Product Button (iAPI) | Same shared state ("N in cart" text, quantity); same nocache rule | Reads `state.cart` reactively, so it updates when the refresh lands. Adds through the shared store's batch queue. |
| Add to Cart + Options, Product Collection, Product Filters, Product Gallery | No cart reads on catalogue pages | Safe. |
| Cart and Checkout (React) | Preloads `/wc/store/v1/cart` (and `/checkout`, saved payment methods) into `wp.apiFetch.createPreloadingMiddleware(…)`, response headers included (`Nonce`, `Cart-Token`) | Fine because those pages are never cached. |
| All Products (legacy React) | Preloads `/wc/store/v1/cart` with headers on any page | Don't use on cached pages; use Product Collection. |
| `wp-api-fetch` (core) | Inline `wp_rest` nonce via `createNonceMiddleware` | For guests it is the user-0 nonce and grants nothing, but don't build on it. |

## Render-time traps

- `WC()->cart`, `WC()->session`, `WC()->customer`, `wp_create_nonce()`, `is_user_logged_in()` or the current user in a template, pattern, `render.php` or `render_callback` of a cacheable page. Whatever the first visitor gets is served to everyone.
- **Prices and location.** `get_price_html()` and the hydrated empty-cart body depend on the customer's tax location. Set *Default customer location* (`woocommerce_default_customer_address`) to "Shop country/region" or "No location by default". "Geolocate (with page caching support)" redirects with a `?v=<hash>` query (`WC_Cache_Helper::geolocation_ajax_redirect()`), which fragments the cache; plain "Geolocate" can make prices, and plausibly the country in the hydrated cart state (**untested**), differ per visitor.
- **Session notices.** The Store Notices block prints `woocommerce_output_all_notices()` at render time. Anything that calls `wc_add_notice()` for a guest (classic `?add-to-cart=` links, `WC()->cart->apply_coupon()` in your own code) makes the next rendered page carry that visitor's notice. The Store API routes do not add session notices; use them.
- **Integration data and inline config.** Anything you pass to the page must be site-wide (currency, thresholds, translated strings, REST root): `script_module_data_{$id}`, `wp_interactivity_config()`, `wp_add_inline_script()`, and `IntegrationInterface::get_script_data()`, which lands in inline `wcSettings` (a Mini-Cart integration prints it on every cached page).
- **Coming-soon mode** (`woocommerce_coming_soon = yes`). Visitors without `manage_woocommerce` get the coming-soon template with `Cache-Control: max-age=60`; with "store pages only" (`woocommerce_store_pages_only`) only shop pages are hidden. Exclusions: filter `woocommerce_coming_soon_exclude`, or the private `?woo-share=<key>` link (sets a 90-day `woo-share` cookie). Toggling either option only runs `wp_cache_flush()`: purge the page/edge cache yourself on launch, and bypass `woo-share` cookies.

## Purging the page cache from Action Scheduler

Never call a CDN API inside a save or stock hook (stock changes fire during checkout). Queue product ids, and let one unique Action Scheduler job purge the batch.

```php
<?php
/**
 * Batched edge purge for product changes. my_plugin_cdn_purge() is the CDN-specific part.
 */
add_action( 'woocommerce_update_product', 'my_plugin_queue_product_purge' );
add_action( 'woocommerce_update_product_variation', 'my_plugin_queue_product_purge' );
add_action( 'woocommerce_product_set_stock', 'my_plugin_queue_product_purge' );   // Passes a WC_Product.
add_action( 'woocommerce_variation_set_stock', 'my_plugin_queue_product_purge' );
add_action( 'my_plugin_purge_products', 'my_plugin_run_product_purge' );
add_action( 'update_option_woocommerce_coming_soon', 'my_plugin_purge_everything' );

function my_plugin_queue_product_purge( $product ) {
	$product = $product instanceof WC_Product ? $product : wc_get_product( $product );
	if ( ! $product ) {
		return;
	}
	$queue = (array) get_option( 'my_plugin_purge_queue', array() );
	$queue[ $product->get_parent_id() ? $product->get_parent_id() : $product->get_id() ] = true;
	update_option( 'my_plugin_purge_queue', $queue, false );

	// Unique = same hook + args + group not already pending or running. The delay batches bursts of saves.
	as_schedule_single_action( time() + 30, 'my_plugin_purge_products', array(), 'my-plugin', true );
}

function my_plugin_run_product_purge() {
	$queue = (array) get_option( 'my_plugin_purge_queue', array() );
	delete_option( 'my_plugin_purge_queue' );

	$urls = array( home_url( '/' ), wc_get_page_permalink( 'shop' ) );
	foreach ( array_keys( $queue ) as $product_id ) {
		$urls[] = get_permalink( $product_id );
		$terms  = get_the_terms( $product_id, 'product_cat' );
		foreach ( is_array( $terms ) ? $terms : array() as $term ) {
			$urls[] = is_wp_error( get_term_link( $term ) ) ? '' : get_term_link( $term );
		}
	}
	foreach ( array_chunk( array_values( array_unique( array_filter( $urls ) ) ), 30 ) as $files ) {
		my_plugin_cdn_purge( array( 'files' => $files ) );
	}

	// Ids queued while this job ran could not schedule a unique job (this one counts as running).
	if ( get_option( 'my_plugin_purge_queue' ) ) {
		as_schedule_single_action( time() + 30, 'my_plugin_purge_products', array(), 'my-plugin' );
	}
}

function my_plugin_purge_everything() {
	my_plugin_cdn_purge( array( 'purge_everything' => true ) );
}

/** Cloudflare: MY_PLUGIN_CF_ZONE_ID + MY_PLUGIN_CF_API_TOKEN (Cache Purge permission) in wp-config.php. */
function my_plugin_cdn_purge( array $body ) {
	if ( ! defined( 'MY_PLUGIN_CF_ZONE_ID' ) || ! defined( 'MY_PLUGIN_CF_API_TOKEN' ) ) {
		return;
	}
	$response = wp_remote_post(
		'https://api.cloudflare.com/client/v4/zones/' . rawurlencode( MY_PLUGIN_CF_ZONE_ID ) . '/purge_cache',
		array(
			'timeout' => 15,
			'headers' => array(
				'Authorization' => 'Bearer ' . MY_PLUGIN_CF_API_TOKEN,
				'Content-Type'  => 'application/json',
			),
			'body'    => wp_json_encode( $body ),
		)
	);
	if ( is_wp_error( $response ) || 200 !== wp_remote_retrieve_response_code( $response ) ) {
		// Marks the action failed; Action Scheduler does not retry failed actions.
		throw new RuntimeException( 'Cache purge failed.' );
	}
}
```

The option queue is a read-modify-write: two simultaneous saves can drop an id. Acceptable for cache purging; use a custom table if you need exactness.

## Pattern: a Store API client for cached pages

A dependency-free script module. It reads the REST root from core's script-module data (`script_module_data_{$module_id}`, printed for enqueued modules and their dependencies), keeps the nonce from responses, warms it with a `GET /cart` before the first write, and retries once on a nonce error. It persists under WooCommerce's own key and JSON shape so `wc-blocks-middleware` can read it on the next page.

```php
add_action( 'init', 'my_plugin_register_store_api_modules' );
add_filter( 'script_module_data_my-plugin/store-api', 'my_plugin_store_api_module_data' );

function my_plugin_register_store_api_modules() {
	$base = plugin_dir_url( __FILE__ ) . 'assets/js/';
	wp_register_script_module( 'my-plugin/store-api', $base . 'store-api.js', array(), '1.0.0' );
	wp_register_script_module( 'my-plugin/add-to-cart', $base . 'add-to-cart.js', array( 'my-plugin/store-api' ), '1.0.0' );
	wp_register_script_module( 'my-plugin/free-shipping', $base . 'free-shipping.js', array( 'my-plugin/store-api' ), '1.0.0' );
}

function my_plugin_store_api_module_data( array $data ) {
	$data['restUrl'] = esc_url_raw( rest_url() ); // Site-wide, cache-safe.
	return $data;
}
```

```js
// assets/js/store-api.js — module id: my-plugin/store-api
const NONCE_KEY = 'storeApiNonce'; // Same key and { nonce, timestamp } shape as wc-blocks-middleware.

export function readModuleData( id ) {
	try {
		const el = document.getElementById( `wp-script-module-data-${ id }` );
		return el ? JSON.parse( el.textContent ) : {};
	} catch ( e ) {
		return {};
	}
}

const { restUrl = '/wp-json/' } = readModuleData( 'my-plugin/store-api' );

function endpoint( path ) {
	const url = new URL( restUrl, window.location.origin );
	if ( url.searchParams.has( 'rest_route' ) ) {
		url.searchParams.set( 'rest_route', `/wc/store/v1/${ path }` ); // Plain permalinks.
	} else {
		url.pathname = `${ url.pathname.replace( /\/?$/, '/' ) }wc/store/v1/${ path }`;
	}
	return url.toString();
}

let pageNonce = ''; // Only a nonce received during this page view is trusted for writes.
let warmup = null;

function rememberNonce( response ) {
	const nonce = response.headers.get( 'Nonce' );
	if ( ! nonce ) {
		return;
	}
	pageNonce = nonce;
	const timestamp = Number( response.headers.get( 'Nonce-Timestamp' ) ) || Math.floor( Date.now() / 1000 );
	try {
		const stored = JSON.parse( window.localStorage.getItem( NONCE_KEY ) || '{}' ) || {};
		if ( ! stored.timestamp || timestamp >= Number( stored.timestamp ) ) {
			window.localStorage.setItem( NONCE_KEY, JSON.stringify( { nonce, timestamp } ) );
		}
	} catch ( e ) {
		// Storage blocked: the in-memory nonce is enough for this page.
	}
}

async function send( path, method, body ) {
	const headers = { Accept: 'application/json' };
	if ( undefined !== body ) {
		headers[ 'Content-Type' ] = 'application/json';
	}
	if ( 'GET' !== method ) {
		headers.Nonce = pageNonce;
	}
	const response = await fetch( endpoint( path ), {
		method,
		headers,
		body: undefined === body ? undefined : JSON.stringify( body ),
		credentials: 'same-origin', // Sends the HttpOnly wp_woocommerce_session_* cookie.
		cache: 'no-store',
	} );
	rememberNonce( response );
	return response;
}

function ensureNonce() {
	if ( pageNonce ) {
		return Promise.resolve();
	}
	if ( ! warmup ) {
		warmup = send( 'cart', 'GET' ).finally( () => {
			warmup = null;
		} );
	}
	return warmup;
}

export async function request( path, { method = 'GET', body } = {} ) {
	if ( 'GET' !== method ) {
		await ensureNonce();
	}
	let response = await send( path, method, body );

	if ( 'GET' !== method && ( 401 === response.status || 403 === response.status ) ) {
		const error = await response.clone().json().catch( () => ( {} ) );
		if ( [ 'woocommerce_rest_invalid_nonce', 'woocommerce_rest_missing_nonce' ].includes( error.code ) ) {
			pageNonce = '';
			await ensureNonce();
			response = await send( path, method, body );
		}
	}

	const data = await response.json().catch( () => null );
	if ( ! response.ok ) {
		const { message = response.statusText, code = 'unknown_error', data: extra } = data || {};
		throw Object.assign( new Error( message ), { code, status: response.status, data: extra } );
	}
	return data;
}

export const getCart = () => request( 'cart' );

export const addItem = ( id, quantity = 1, variation = [] ) =>
	request( 'cart/add-item', {
		method: 'POST',
		body: { id, quantity, ...( variation.length ? { variation } : {} ) },
	} );

export const cartExtensionUpdate = ( namespace, data ) =>
	request( 'cart/extensions', { method: 'POST', body: { namespace, data } } );

export const hasItemsCookie = () =>
	document.cookie.split( ';' ).some( ( c ) => c.trim().startsWith( 'woocommerce_items_in_cart=' ) );

// Public contract: the Mini-Cart (and React cart stores) refresh on this event.
export function notifyCartChanged() {
	document.body.dispatchEvent(
		new CustomEvent( 'wc-blocks_added_to_cart', {
			bubbles: true,
			cancelable: true,
			detail: { preserveCartData: false }, // false → React stores refetch too.
		} )
	);
}

// Store API messages are HTML-escaped strings; decode before using textContent.
export const plainText = ( html ) =>
	new DOMParser().parseFromString( String( html ), 'text/html' ).body.textContent || '';
```

If a page already loads `wp-api-fetch` + `wc-blocks-middleware`, `wp.apiFetch` attaches the nonce for you, but the seed bug above still applies: make a Store API GET before the first write.

## Custom shopper endpoints under `wc/store/v1`

When `extensions` data and `cart/extensions` are not enough, register your own route **in the `wc/store/v1` namespace**, never `admin-ajax`, `wc-ajax` or a separate namespace. Any request whose route starts with `/wc/store/` is a Store API request for core (`WooCommerce::is_store_api_request()`, and `StoreApi\Authentication` hooks on `rest_route` starting with `/wc/store/`), so it gets the Store API session handler (cookie, or Cart-Token), the Store API CORS headers (`Nonce`, `Cart-Token` allowed and exposed), and its rate limiting. What it does **not** get, compared with core cart routes:

- **No cart loaded.** WooCommerce only calls `wc_load_cart()` on frontend requests; Store API routes load it themselves. Do the same before touching `WC()->cart`/`WC()->session`, and only in per-shopper callbacks.
- **No nonce check.** For `/wc/store/` requests `Authentication::check_authentication()` returns `true` on `rest_authentication_errors`, so WordPress's `wp_rest` cookie-nonce check never runs and a logged-in cookie is honoured without it. A write route must verify the `Nonce` header (`wc_store_api`) itself, or it is CSRF-able.
- **No `Nonce`/`Cache-Control` headers.** Send `Cache-Control: no-store` (plus `Nonce`/`Nonce-Timestamp`, so the shared client keeps a fresh nonce) on per-shopper responses, and a short public `max-age` on public ones. WordPress itself sends no-cache headers only for logged-in users (`rest_send_nocache_headers`).
- A public route must not load the cart or session: that is what keeps it cacheable and cookie-free. With the bypass rules above the edge still sends all of `/wp-json/` to origin, so `max-age` only helps the browser unless you add a dedicated edge rule that respects `Cache-Control`.

```php
add_action( 'rest_api_init', 'my_plugin_register_store_routes' );

function my_plugin_register_store_routes() {
	// Per-shopper: read and write a gift note kept in the WooCommerce session.
	register_rest_route(
		'wc/store/v1',
		'/my-plugin/gift-note',
		array(
			array(
				'methods'             => WP_REST_Server::READABLE,
				'callback'            => 'my_plugin_get_gift_note',
				'permission_callback' => '__return_true', // Public route; the session decides whose data it is.
			),
			array(
				'methods'             => WP_REST_Server::CREATABLE,
				'callback'            => 'my_plugin_update_gift_note',
				'permission_callback' => 'my_plugin_check_store_api_nonce',
				'args'                => array(
					'note' => array(
						'description'       => __( 'Gift note printed on the packing slip.', 'my-plugin' ),
						'type'              => 'string',
						'required'          => true,
						'maxLength'         => 200,
						// A custom sanitize_callback disables WP's default type/maxLength check, so validate explicitly.
						'validate_callback' => 'rest_validate_request_arg',
						'sanitize_callback' => 'sanitize_textarea_field',
					),
				),
			),
			'schema' => 'my_plugin_gift_note_schema',
		)
	);

	// Public: identical for every visitor, no session, browser-cacheable for a short time.
	register_rest_route(
		'wc/store/v1',
		'/my-plugin/stock/(?P<id>\d+)',
		array(
			'methods'             => WP_REST_Server::READABLE,
			'callback'            => 'my_plugin_get_stock',
			'permission_callback' => '__return_true',
			'args'                => array(
				'id' => array(
					'type'     => 'integer', // No sanitize_callback: WP validates and casts from the schema.
					'minimum'  => 1,
					'required' => true,
				),
			),
		)
	);
}

function my_plugin_gift_note_schema() {
	return array(
		'$schema'    => 'http://json-schema.org/draft-04/schema#',
		'title'      => 'my_plugin_gift_note',
		'type'       => 'object',
		'properties' => array(
			'note'        => array( 'type' => 'string' ),
			'items_count' => array( 'type' => 'integer' ),
		),
	);
}

function my_plugin_check_store_api_nonce( WP_REST_Request $request ) {
	$nonce = (string) $request->get_header( 'Nonce' );
	if ( '' === $nonce || ! wp_verify_nonce( $nonce, 'wc_store_api' ) ) {
		// Same codes as core, so the shared client's retry-once logic applies.
		return new WP_Error(
			'' === $nonce ? 'woocommerce_rest_missing_nonce' : 'woocommerce_rest_invalid_nonce',
			__( 'Nonce is invalid.', 'my-plugin' ),
			array( 'status' => '' === $nonce ? 401 : 403 )
		);
	}
	return true;
}

/**
 * Loads session + cart the way core Store API routes do. Only call it in per-shopper callbacks.
 */
function my_plugin_load_store_cart() {
	if ( ! did_action( 'woocommerce_load_cart_from_session' ) ) {
		wc_load_cart();
	}
	WC()->cart->get_cart(); // Reads the cart from the session.
}

function my_plugin_shopper_response( array $data ) {
	$response = rest_ensure_response( $data );
	$response->header( 'Cache-Control', 'no-store' );
	$response->header( 'Nonce', wp_create_nonce( 'wc_store_api' ) );
	$response->header( 'Nonce-Timestamp', (string) time() );
	return $response;
}

function my_plugin_get_gift_note() {
	my_plugin_load_store_cart();
	return my_plugin_shopper_response(
		array(
			'note'        => (string) WC()->session->get( 'my_plugin_gift_note', '' ),
			'items_count' => (int) WC()->cart->get_cart_contents_count(),
		)
	);
}

function my_plugin_update_gift_note( WP_REST_Request $request ) {
	my_plugin_load_store_cart();
	if ( WC()->cart->is_empty() ) {
		return new WP_Error( 'my_plugin_empty_cart', __( 'Add a product before writing a gift note.', 'my-plugin' ), array( 'status' => 409 ) );
	}
	// A guest session is only persisted once its cookie exists (it does whenever the cart has items).
	WC()->session->set_customer_session_cookie( true );
	WC()->session->set( 'my_plugin_gift_note', $request['note'] );

	return my_plugin_shopper_response(
		array(
			'note'        => $request['note'],
			'items_count' => (int) WC()->cart->get_cart_contents_count(),
		)
	);
}

function my_plugin_get_stock( WP_REST_Request $request ) {
	$product = wc_get_product( $request['id'] );
	if ( ! $product || 'publish' !== $product->get_status() ) {
		return new WP_Error( 'my_plugin_not_found', __( 'Product not found.', 'my-plugin' ), array( 'status' => 404 ) );
	}
	$response = rest_ensure_response(
		array(
			'id'           => $product->get_id(),
			'in_stock'     => $product->is_in_stock(),
			'stock_status' => $product->get_stock_status(),
		)
	);
	$response->header( 'Cache-Control', 'public, max-age=30' );
	return $response;
}
```

From the browser, use the same client (it adds the nonce, retries once on `woocommerce_rest_invalid_nonce`, and picks up the `Nonce` header these responses send):

```js
import { request } from 'my-plugin/store-api';

const { note } = await request( 'my-plugin/gift-note' ); // GET, per shopper.
await request( 'my-plugin/gift-note', { method: 'POST', body: { note: 'Happy birthday!' } } );
const stock = await request( 'my-plugin/stock/123' ); // Public.
```

## Pattern: custom add-to-cart button that refreshes the Mini-Cart

Prefer the Product Button or Add to Cart + Options blocks. When a design needs its own button, render markup that only depends on the product, and post from JS.

```php
/**
 * Cache-safe add-to-cart button: depends on the product only, never on the cart.
 */
function my_plugin_add_to_cart_button( WC_Product $product ) {
	if ( ! $product->is_purchasable() || ! $product->is_in_stock() || ! $product->is_type( 'simple' ) ) {
		return '';
	}
	wp_enqueue_script_module( 'my-plugin/add-to-cart' );

	return sprintf(
		'<div class="my-plugin-atc"><button type="button" class="wp-element-button" data-my-plugin-add-to-cart data-product-id="%1$d" data-success-text="%2$s">%3$s</button><span class="screen-reader-text" role="status"></span></div>',
		absint( $product->get_id() ),
		/* translators: %s: product name. */
		esc_attr( sprintf( __( '%s was added to your cart.', 'my-plugin' ), $product->get_name() ) ),
		esc_html__( 'Add to cart', 'my-plugin' )
	);
}
```

```js
// assets/js/add-to-cart.js — module id: my-plugin/add-to-cart
import { addItem, notifyCartChanged, plainText } from 'my-plugin/store-api';

document.addEventListener( 'click', async ( event ) => {
	const button = event.target.closest( '[data-my-plugin-add-to-cart]' );
	if ( ! button || button.disabled ) {
		return;
	}
	const status = button.parentElement.querySelector( '[role="status"]' );
	button.disabled = true;
	button.setAttribute( 'aria-busy', 'true' );
	try {
		await addItem( Number( button.dataset.productId ), 1 );
		notifyCartChanged();
		status.textContent = button.dataset.successText;
	} catch ( error ) {
		status.textContent = plainText( error.message ); // Already translated by the server.
	} finally {
		button.disabled = false;
		button.removeAttribute( 'aria-busy' );
	}
} );
```

## Pattern: "free shipping in $X" from client-side totals

The threshold and copy are site-wide, so they go into the page; the cart amounts come from the Store API. Amounts in the cart response are strings in minor units, with `currency_minor_unit` and the store's separators, prefix and suffix alongside. Compare the same amount your shipping rule compares (this example uses items subtotal including tax).

```php
add_filter( 'script_module_data_my-plugin/free-shipping', 'my_plugin_free_shipping_module_data' );

function my_plugin_free_shipping_module_data( array $data ) {
	$threshold = (float) get_option( 'my_plugin_free_shipping_threshold', 0 );

	$data['threshold'] = (string) absint( round( $threshold * pow( 10, wc_get_price_decimals() ) ) );
	/* translators: %s: amount left for free shipping. */
	$data['remaining'] = __( 'Add %s more for free shipping.', 'my-plugin' );
	$data['reached']   = __( 'Your order ships free.', 'my-plugin' );
	return $data;
}
```

```js
// assets/js/free-shipping.js — module id: my-plugin/free-shipping
// Markup anywhere in the template: <p data-my-plugin-free-shipping role="status" hidden></p>
import { getCart, hasItemsCookie, readModuleData } from 'my-plugin/store-api';

const el = document.querySelector( '[data-my-plugin-free-shipping]' );
const config = readModuleData( 'my-plugin/free-shipping' );

function formatMoney( minor, t ) {
	const unit = t.currency_minor_unit;
	const [ int, dec ] = ( Number( minor ) / 10 ** unit ).toFixed( unit ).split( '.' );
	const grouped = int.replace( /\B(?=(\d{3})+(?!\d))/g, t.currency_thousand_separator );
	return t.currency_prefix + grouped + ( dec ? t.currency_decimal_separator + dec : '' ) + t.currency_suffix;
}

async function render( force = false ) {
	if ( ! el || ! Number( config.threshold ) ) {
		return;
	}
	if ( ! force && ! hasItemsCookie() ) {
		el.hidden = true; // Same gate core uses: no items cookie → empty cart, no request.
		return;
	}
	const { totals, items_count: count } = await getCart();
	const subtotal = Number( totals.total_items ) + Number( totals.total_items_tax );
	const left = Number( config.threshold ) - subtotal;
	el.hidden = 0 === count;
	el.textContent = left > 0 ? config.remaining.replace( '%s', formatMoney( left, totals ) ) : config.reached;
}

render().catch( () => {} );
document.body.addEventListener( 'wc-blocks_added_to_cart', () => render( true ).catch( () => {} ) );
document.body.addEventListener( 'wc-blocks_removed_from_cart', () => render( true ).catch( () => {} ) );
// Removals inside the iAPI Mini-Cart emit only this internal sync event (not a documented contract).
window.addEventListener( 'wc-blocks_store_sync_required', ( e ) => {
	if ( 'from_iAPI' === e.detail?.type ) {
		render( true ).catch( () => {} );
	}
} );
```

## Extending Store API responses (ExtendSchema)

`woocommerce_store_api_register_endpoint_data()` adds data under `extensions.<namespace>` of four schemas only: `CartItemSchema::IDENTIFIER` (`cart-item`), `CartSchema::IDENTIFIER` (`cart`), `CheckoutSchema::IDENTIFIER` (`checkout`) and `ProductSchema::IDENTIFIER` (`product`). `data_callback` receives the cart item array (cart-item), the `WC_Product` (product), or nothing (cart, checkout). Register on `woocommerce_blocks_loaded`, which fires on every request including Store API ones. The function returns `true|WP_Error`; registration errors only throw when `WP_DEBUG` is on and the user can `manage_woocommerce`.

```php
use Automattic\WooCommerce\StoreApi\Schemas\V1\CartItemSchema;
use Automattic\WooCommerce\StoreApi\Schemas\V1\CartSchema;
use Automattic\WooCommerce\StoreApi\Schemas\V1\ProductSchema;

add_action( 'woocommerce_blocks_loaded', 'my_plugin_extend_store_api' );

function my_plugin_extend_store_api() {
	$field = static function ( $type, $description ) {
		return array(
			'description' => $description,
			'type'        => $type,
			'context'     => array( 'view', 'edit' ),
			'readonly'    => true,
		);
	};

	woocommerce_store_api_register_endpoint_data(
		array(
			'endpoint'        => CartItemSchema::IDENTIFIER,
			'namespace'       => 'my-plugin',
			'data_callback'   => static function ( array $cart_item ) {
				$per_unit = max( 1, (int) $cart_item['data']->get_meta( '_my_plugin_bottles' ) );
				return array( 'bottles' => $per_unit * (int) $cart_item['quantity'] );
			},
			'schema_callback' => static function () use ( $field ) {
				return array( 'bottles' => $field( 'integer', __( 'Bottles this line adds.', 'my-plugin' ) ) );
			},
			'schema_type'     => ARRAY_A,
		)
	);

	woocommerce_store_api_register_endpoint_data(
		array(
			'endpoint'        => CartSchema::IDENTIFIER,
			'namespace'       => 'my-plugin',
			// Runs on every cart response: keep it cheap, no remote calls.
			'data_callback'   => static function () {
				return array( 'gift_wrap' => (bool) ( WC()->session && WC()->session->get( 'my_plugin_gift_wrap' ) ) );
			},
			'schema_callback' => static function () use ( $field ) {
				return array( 'gift_wrap' => $field( 'boolean', __( 'Whether gift wrap is selected.', 'my-plugin' ) ) );
			},
			'schema_type'     => ARRAY_A,
		)
	);

	woocommerce_store_api_register_endpoint_data(
		array(
			'endpoint'        => ProductSchema::IDENTIFIER,
			'namespace'       => 'my-plugin',
			'data_callback'   => static function ( WC_Product $product ) {
				return array( 'vintage' => (string) $product->get_meta( '_my_plugin_vintage' ) );
			},
			'schema_callback' => static function () use ( $field ) {
				return array( 'vintage' => $field( 'string', __( 'Vintage year.', 'my-plugin' ) ) );
			},
			'schema_type'     => ARRAY_A,
		)
	);
}
```

Cart-item and cart extension data reaches the iAPI store too (`state.cart` is the `/cart` body), and the React stores. Values posted under `extensions` (checkout) are validated and sanitized recursively from the schema and **undeclared keys are dropped**; see `checkout-blocks.md`.

## Cart updates from JS: update callback + `extensionCartUpdate`

`woocommerce_store_api_register_update_callback()` serves exactly one route, `POST /wc/store/v1/cart/extensions` `{ namespace, data }`. Your callback runs, then totals are recalculated and the cart is returned. Exceptions are not caught around the callback: throw `RouteException` for a clean 4xx. It is not a checkout validator.

```php
use Automattic\WooCommerce\StoreApi\Exceptions\RouteException;

add_action( 'woocommerce_blocks_loaded', 'my_plugin_register_gift_wrap_callback' );
add_action( 'woocommerce_cart_calculate_fees', 'my_plugin_add_gift_wrap_fee' );

function my_plugin_register_gift_wrap_callback() {
	woocommerce_store_api_register_update_callback(
		array(
			'namespace' => 'my-plugin-gift-wrap',
			'callback'  => 'my_plugin_update_gift_wrap',
		)
	);
}

function my_plugin_update_gift_wrap( $data ) {
	if ( ! is_array( $data ) || ! isset( $data['enabled'] ) ) {
		throw new RouteException( 'my_plugin_invalid_gift_wrap', esc_html__( 'Invalid gift wrap request.', 'my-plugin' ), 400 );
	}
	WC()->session->set( 'my_plugin_gift_wrap', (bool) $data['enabled'] );
}

function my_plugin_add_gift_wrap_fee( WC_Cart $cart ) {
	if ( WC()->session && WC()->session->get( 'my_plugin_gift_wrap' ) ) {
		// Amount in major units of the store currency (not minor units).
		$cart->add_fee( __( 'Gift wrap', 'my-plugin' ), 2990, true );
	}
}
```

Client side: inside Cart/Checkout blocks use `extensionCartUpdate( { namespace: 'my-plugin-gift-wrap', data: { enabled: true } } )` from `@woocommerce/blocks-checkout` (it updates `wc/store/cart` with the response). On cached pages use the client above, `await cartExtensionUpdate( 'my-plugin-gift-wrap', { enabled: true } )`, then `notifyCartChanged()`.

## Fees and discounts

- `woocommerce_cart_calculate_fees` works in the block cart and checkout: the Store API runs `WC_Cart::calculate_totals()` on cart routes, `cart/extensions`, `PUT /checkout?__experimental_calc_totals=true` (sent when payment method, notes or additional fields change) and `POST /checkout`. Fees appear in `cart.fees` and `totals.total_fees`. Drop the classic `is_admin() && ! DOING_AJAX` guard.
- Payment-method fees update live: read `WC()->session->get( 'chosen_payment_method' )` in the fee callback; the Checkout block re-PUTs on method change.
- Discounts: `add_fee()` says not to pass negative amounts (totals clamp negative fees). Use coupons. A virtual coupon via `woocommerce_get_shop_coupon_data` needs no database row; apply it from JS with `POST /wc/store/v1/cart/apply-coupon { "code": "…" }`, which validates without adding session notices. `WC()->cart->apply_coupon()` adds notices (see Render-time traps).

```php
add_filter( 'woocommerce_get_shop_coupon_data', 'my_plugin_virtual_club_coupon', 10, 2 );

function my_plugin_virtual_club_coupon( $data, $code ) {
	if ( 'my-plugin-club' !== wc_format_coupon_code( (string) $code ) ) {
		return $data;
	}
	return array(
		'discount_type' => 'percent',
		'amount'        => 10,
	);
}
```

## Add-to-cart and cart validation

| Need | Hook | How to reject |
|---|---|---|
| Block an add-to-cart | action `woocommerce_store_api_validate_add_to_cart( WC_Product $product, array $request )` | throw `RouteException` |
| Reject a quantity change of an item already in the cart (11.2) | filter `woocommerce_store_api_cart_item_quantity_validation( true, $quantity, $product, $cart_item )` | return `WP_Error` (does not run on first add) |
| Min / max / step | filters `woocommerce_store_api_product_quantity_{minimum,maximum,multiple_of}( $value, $product, $cart_item )` | return the number |
| Per item, every cart validation | action `woocommerce_store_api_validate_cart_item( WC_Product $product, array $cart_item )` | throw |
| Whole cart (also runs before checkout) | action `woocommerce_store_api_cart_errors( WP_Error $errors, WC_Cart $cart )` | add to `$errors` → 409 |

```php
use Automattic\WooCommerce\StoreApi\Exceptions\RouteException;

add_action( 'woocommerce_store_api_validate_add_to_cart', 'my_plugin_limit_per_order', 10, 2 );
add_action( 'woocommerce_store_api_cart_errors', 'my_plugin_require_box_multiples', 10, 2 );

function my_plugin_limit_per_order( WC_Product $product, array $request ) {
	$limit = (int) $product->get_meta( '_my_plugin_limit' );
	if ( $limit > 0 && (int) $request['quantity'] > $limit ) {
		throw new RouteException(
			'my_plugin_limit_exceeded',
			/* translators: %d: maximum quantity. */
			esc_html( sprintf( __( 'You can buy up to %d of this product.', 'my-plugin' ), $limit ) ),
			400
		);
	}
}

function my_plugin_require_box_multiples( WP_Error $errors, WC_Cart $cart ) {
	if ( 0 !== $cart->get_cart_contents_count() % 6 ) {
		$errors->add( 'my_plugin_box_multiple', __( 'Wines are sold in boxes of 6.', 'my-plugin' ) );
	}
}
```

The Store API still applies the deprecated `woocommerce_add_to_cart_validation` filter; don't write new code against it, and don't use `wc_add_notice()` to report errors.

## Extending the Interactivity API blocks

The shared stores (`woocommerce`, `woocommerce/store-notices`, …) and almost every WooCommerce block store are registered with a `lock` ("I acknowledge that using a private store means my plugin will inevitably break…"); the PHP side, `BlocksSharedState::load_cart_state()`, demands a similar consent string. Using them is opting into breakage. The supported surfaces:

1. **DOM events.** Dispatch `wc-blocks_added_to_cart` / `wc-blocks_removed_from_cart` on `document.body` (bubbling) after you change the cart; listen to them to react. Your own iAPI block can subscribe declaratively: `data-wp-on-document--wc-blocks_added_to_cart="actions.refresh"`.
2. **Store API data.** ExtendSchema output appears in `state.cart.items[].extensions` without touching the store.
3. **PHP render filters.** `render_block_woocommerce/<block>`, Block Hooks, and for the Product Button `woocommerce_loop_add_to_cart_args` (`class`, `attributes`; since 9.7) and `woocommerce_loop_add_to_cart_link` (final HTML).
4. **Scripts next to the Mini-Cart.** Register an `IntegrationInterface` on `woocommerce_blocks_mini-cart_block_registration`; the Mini-Cart enqueues its handles.

```php
add_filter( 'woocommerce_loop_add_to_cart_args', 'my_plugin_product_button_args', 10, 2 );

function my_plugin_product_button_args( array $args, WC_Product $product ) {
	// Product-only data: cache-safe.
	$args['attributes']['data-my-plugin-vintage'] = (string) $product->get_meta( '_my_plugin_vintage' );
	return $args;
}
```

## Common mistakes

- Printing the cart count, mini-cart items, customer name or a nonce into HTML of a cacheable page, or reading `WC()->cart` in a render callback.
- Building a header cart with cart fragments, `admin-ajax` or `wc-ajax`; use the Mini-Cart block or the Store API.
- Sending a write with a nonce taken from HTML or `localStorage` on a cached page without a fresh Store API response first; no retry on `403 woocommerce_rest_invalid_nonce`.
- Persisting a `Cart-Token` (or a nonce under a custom key, or in a format other than `{ nonce, timestamp }` JSON under `storeApiNonce`).
- `fetch()` to the Store API without `credentials: 'same-origin'` from a different origin/subdomain, so the session cookie is missing and every call sees an empty cart.
- Bypassing the cache on `woocommerce_items_in_cart`/session cookies "to be safe" and losing the cache for every shopper; or the opposite, an edge rule that caches responses with `Set-Cookie`/`no-store`.
- Caching `/wp-json/` at the edge, or a browser TTL on HTML that outlives a purge.
- Shopper endpoints in `admin-ajax`, `wc-ajax` or a separate REST namespace instead of `wc/store/v1/my-plugin/…`; or a write route there without its own `Nonce` check, `wc_load_cart()` or `Cache-Control: no-store`.
- Calling a CDN purge API synchronously from `woocommerce_update_product` or a stock hook.
- Using the All Products block on cached pages (it preloads the cart with headers).
- Geolocated default customer location on a cached shop.
- Validating checkout data with `woocommerce_store_api_register_update_callback`, or throwing a plain `Exception` in it (500).
- Negative fees for discounts; `WC()->cart->apply_coupon()` from Store API code (session notices leak into the next rendered page).
- `store( 'woocommerce', …, { lock } )` in production code to read the cart: subscribe to the events or call the Store API.
