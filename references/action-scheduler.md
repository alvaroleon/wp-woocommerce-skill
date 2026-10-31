# Action Scheduler

All background and scheduled work in a WooCommerce plugin goes through Action Scheduler (AS): order syncs, stock imports, webhook processing, delayed emails, cleanups. WooCommerce 11.2.1 bundles **AS 4.1.0** (`packages/action-scheduler`). This file covers the API, the exact semantics that bite (uniqueness, args, failures), how the queue actually gets run, and two complete production patterns.

## The premise, and its nuance

**Never `wp_schedule_event()` / `wp_schedule_single_event()` / `cron_schedules` in plugin code.** AS gives you persistence, uniqueness, groups, logs, a failure state, an admin UI (Tools › Scheduled Actions, also WooCommerce › Status › Scheduled Actions) and WP-CLI. WooCommerce schedules its own recurring maintenance in AS. Some core code still uses WP-Cron (background updater, background emailer, admin notes, stock-notification retention), so the site still needs WP-Cron to run.

The nuance: **AS's default runner is itself triggered by WP-Cron.**

- `ActionScheduler_QueueRunner::init()` (on `init` priority 1) registers the WP-Cron event `action_scheduler_run_queue` on an `every_minute` schedule and hooks `run()` to it. Each run processes batches of 25 (`action_scheduler_queue_runner_batch_size`) for up to 30 s (`action_scheduler_queue_runner_time_limit`), one concurrent batch by default (`action_scheduler_queue_runner_concurrent_batches`).
- **Async request runner:** on `shutdown` of **admin** requests (`is_admin()`), at most once per 60 s, AS sends a loopback request that starts a queue run. Filter: `action_scheduler_allow_async_request_runner`.
- Consequence for a full-page-cached shop: front-end hits served by the cache never run the queue, and with `DISABLE_WP_CRON` and no system cron nothing runs at all. On low-traffic sites "async" can mean "much later".

**Production setup: a dedicated worker.** Run `wp action-scheduler run` from a separate process (container, systemd timer, cron) and turn off the in-request triggers:

```bash
# Worker loop: drain the queue, then wait. Run WP-Cron separately for the remaining core users.
while true; do
  wp action-scheduler run --batch-size=50 --batches=20 --quiet
  sleep 20
done
# Every minute, with define( 'DISABLE_WP_CRON', true ) in wp-config.php:
wp cron event run --due-now --quiet
```

Options for `wp action-scheduler run`: `--batch-size`, `--batches` (0 = until empty), `--hooks=a,b`, `--group=g`, `--exclude-groups=a,b`, `--free-memory-on`, `--pause`, `--force`. It uses `ActionScheduler_WPCLI_QueueRunner`, not the WP-Cron hook, so the snippet below doesn't affect it.

```php
<?php
/**
 * mu-plugin: disable AS's in-request runners where a dedicated worker exists.
 *
 * Hook and callback verified against the AS 4.1.0 source: QueueRunner::init() runs on 'init'
 * priority 1 and adds array( ActionScheduler_QueueRunner::instance(), 'run' ) to
 * 'action_scheduler_run_queue' at priority 10. Not run-tested: verify in your stack that
 * actions still complete through the worker only.
 */
if ( getenv( 'MY_PLUGIN_AS_DEDICATED_WORKER' ) ) {
	add_filter( 'action_scheduler_allow_async_request_runner', '__return_false' );

	add_action(
		'init',
		static function () {
			if ( class_exists( 'ActionScheduler_QueueRunner' ) ) {
				remove_action( 'action_scheduler_run_queue', array( ActionScheduler_QueueRunner::instance(), 'run' ) );
			}
		},
		20
	);
}
```

## Functions (AS 4.1.0, `functions.php`)

| Signature | Returns | Use |
|---|---|---|
| `as_enqueue_async_action( $hook, $args = [], $group = '', $unique = false, $priority = 10 )` | action id; `0` if a unique duplicate exists or AS isn't ready | "Do this soon, outside this request" |
| `as_schedule_single_action( $timestamp, $hook, $args = [], $group = '', $unique = false, $priority = 10 )` | id / `0` | One-shot at a UTC timestamp (delays, retries) |
| `as_schedule_recurring_action( $timestamp, $interval_in_seconds, $hook, $args = [], $group = '', $unique = false, $priority = 10 )` | id / `0` | Fixed interval. The next instance is created when one runs. |
| `as_schedule_cron_action( $timestamp, $schedule, $hook, $args = [], $group = '', $unique = false, $priority = 10 )` | id / `0` | Cron expression (`'0 3 * * *'`) for wall-clock schedules |
| `as_unschedule_action( $hook, $args = [], $group = '' )` | id or null | Cancels the **next** matching pending action |
| `as_unschedule_all_actions( $hook, $args = [], $group = '' )` | void | Cancels all matching. `( '', [], 'my-plugin' )` cancels a whole group. |
| `as_has_scheduled_action( $hook, $args = null, $group = '' )` | bool | Pending or running? `null` args = any args. The cheap existence check. |
| `as_next_scheduled_action( $hook, $args = null, $group = '' )` | timestamp, `true` (running or async), or `false` | When it's due |
| `as_get_scheduled_actions( $args = [], $return_format = OBJECT )` | array | Query by `hook`, `args`, `group`, `status`, `date`, `per_page`, `orderby`… |
| `as_supports( string $feature ): bool` | bool | e.g. `'ensure_recurring_actions_hook'` |

Every function checks `ActionScheduler::is_initialized()`. Called before the data store is ready (AS fires `action_scheduler_init` on `init` priority 1), they return `0`, an empty array or nothing, and log `_doing_it_wrong`. Schedule from `init` (default priority) or later, or from `action_scheduler_init`.

## Uniqueness, groups, priority, args

- **Unique** (`$unique = true`): the docblock says "same hook and group", but the DB store inserts with `WHERE NOT EXISTS` on **hook + group + args** among actions that are **pending or running**. Consequences:
  - Per-item args give one pending job **per item**: `[ 123 ]` and `[ 124 ]` are both allowed. That's the natural debounce.
  - **A callback can't reschedule itself with identical args and `unique = true`**: the current action is `running`, so the insert is skipped and returns `0`. Change the args (an attempt counter) or pass `unique = false`.
  - Args are compared as JSON strings: `[ '5' ]` ≠ `[ 5 ]`, and key order matters. Always cast.
- **Groups:** one per plugin (`'my-plugin'`). Groups filter the admin UI, let the worker run or skip them (`--group`, `--exclude-groups`), and make cleanup one call. WooCommerce uses `woocommerce`, `woocommerce-db-updates` and `woocommerce-webhooks`.
- **Priority:** 0–255, lower runs first, default 10. Use it for "orders before reports", not for timing.
- **Args:** JSON-encoded. Up to 191 chars go in the indexed `args` column; longer args go to `extended_args` (max **8000** chars) and the indexed column holds a hash. Over 8000 throws `InvalidArgumentException`. **Pass ids and scalars only**, never objects, payloads or prices, and re-read state inside the callback: the data may have changed since you enqueued.
- **Positional spreading:** AS runs `do_action_ref_array( $hook, array_values( $args ) )`. Keys are dropped and order is what counts. `[ 'order_id' => 5, 'attempt' => 2 ]` reaches `function ( $order_id, $attempt )`, so set `$accepted_args` in `add_action()`.
- **Callbacks must be registered on every request**, the WP-CLI worker included. If the hook has no callback when the action runs, AS throws "no callbacks are registered" and marks it failed. Don't register job callbacks inside `is_admin()` or a front-end-only branch.

## Registering recurring actions

AS fires **`action_scheduler_ensure_recurring_actions`** from a daily recurring action of its own (first run "tomorrow 3am" local time, re-checked at most hourly during queue runs and admin page loads). Re-assert your schedules there with `unique = true`; WooCommerce core does exactly that for its own jobs. Because that hook first fires tomorrow, also schedule on activation, plus a cheap transient-guarded check for code that was deployed without an activation (mu-plugins, images built with the plugin already active):

```php
register_activation_hook( __FILE__, 'my_plugin_register_recurring_actions' );
add_action( 'action_scheduler_ensure_recurring_actions', 'my_plugin_register_recurring_actions' );
add_action( 'admin_init', 'my_plugin_maybe_register_recurring_actions' );

function my_plugin_register_recurring_actions() {
	if ( ! function_exists( 'as_schedule_recurring_action' ) ) {
		return; // WooCommerce (and its bundled AS) is not active.
	}
	as_schedule_recurring_action( time() + MINUTE_IN_SECONDS, 15 * MINUTE_IN_SECONDS, 'my_plugin_stock_sync_tick', array(), 'my-plugin', true );
}

function my_plugin_maybe_register_recurring_actions() {
	if ( false === get_transient( 'my_plugin_recurring_checked' ) ) {
		my_plugin_register_recurring_actions(); // Idempotent thanks to $unique.
		set_transient( 'my_plugin_recurring_checked', 1, HOUR_IN_SECONDS );
	}
}
```

A recurring action whose last 5 runs **all failed** stops rescheduling itself (`action_scheduler_recurring_action_failure_threshold`, override `action_scheduler_recurring_action_is_consistently_failing`). The ensure hook will re-create it the next day; fix the cause first.

## Deactivation and uninstall cleanup

```php
register_deactivation_hook( __FILE__, 'my_plugin_unschedule_all' );
function my_plugin_unschedule_all() {
	if ( function_exists( 'as_unschedule_all_actions' ) ) {
		as_unschedule_all_actions( '', array(), 'my-plugin' ); // Empty hook + group = cancel the whole group.
	}
}
```

Cancelled and completed rows are purged by AS retention, so you don't need to delete them.

## Failure handling

- An action that **throws** (or fatals) is marked **failed**. `action_scheduler_failed_execution( $action_id, Throwable $e, $context )` fires; an invalid action fires `action_scheduler_failed_validation`; fatals are caught and logged on shutdown.
- **No automatic retries for single or async actions.** Failed is terminal. Build retries yourself:
  - Classify errors. Transient errors (network, timeout, 408, 429, 5xx) are retried. Permanent ones (other 4xx, validation, missing data) are not: retrying a bad token only burns rate limit.
  - Retry with `as_schedule_single_action( time() + $delay, $hook, [ $id, $attempt + 1 ], $group, true )` and exponential backoff with a cap. The attempt counter in the args is what makes `unique` accept the reschedule.
  - When you schedule a retry, **return normally**: that attempt completes and the next one is pending. On the final attempt, **throw**, so the action shows as failed in the UI and in `--status=failed` queries.
- Make every job **idempotent**: check "already done" state first (a stored external reference), and send an idempotency key if the remote API supports one.

## Timeouts and retention

| Setting | Default | Filter |
|---|---|---|
| Queue run time limit (per run, not per action) | 30 s | `action_scheduler_queue_runner_time_limit` |
| Running action considered failed after | 300 s | `action_scheduler_failure_period` |
| Claimed-but-not-started actions released after | 300 s | `action_scheduler_timeout_period` |
| Completed/cancelled actions purged after | 1 month | `action_scheduler_retention_period` |
| Failed actions purged after | 3 months | `action_scheduler_retention_period_for_failed` |

Keep each action well under 30 s and far under 300 s. Break big jobs into batches. The runner also stops claiming when memory is near the limit (`action_scheduler_memory_exceeded`).

## Batching with a cursor

One action processes one batch and enqueues the next with a cursor in its args. Rules:

- Use a cursor that doesn't shift while you work: a page number over a stable `orderby => 'ID'` when processing doesn't change the filter. When processing **removes** items from the filtered set (e.g. "not yet exported" → exported), always re-query page 1, or use an id cursor.
- Enqueue the next batch only if this one was full.
- Debounce the starter: if a batch of the previous run is still pending or running (`as_has_scheduled_action( $batch_hook, null, $group )`), skip.
- WooCommerce's `BatchProcessingController` (with retries) is `Internal`: no compatibility promise. Prefer the hand-rolled cursor.

## Debouncing

- **Per item:** `as_enqueue_async_action( 'my_plugin_sync_product', [ $product_id ], 'my-plugin', true )`. Ten saves of the same product while the job is pending produce one job.
- **Global window:** schedule once at `time() + 60` only if nothing is pending (`as_has_scheduled_action( $hook, null, $group )`). Later changes inside the window are absorbed. A production edge-cache purge plugin does exactly this: the first content change schedules a purge at T+60 s, the rest no-op, and 429 or network errors retry with backoff at 60 s, then 120 s.

## Observability

- Admin: Tools › Scheduled Actions (filter by group and status, view per-action logs).
- CLI: `wp action-scheduler action list --group=my-plugin --status=failed --fields=id,hook,args,scheduled_date`, `wp action-scheduler action logs <id>`, `wp action-scheduler status`.
- Hooks: `action_scheduler_before_execute`, `action_scheduler_after_execute( $action_id, $action, $context )`, `action_scheduler_failed_execution`.
- Log through `wc_get_logger()` with `source => 'my-plugin'`, and with ids in the message.

```php
add_action( 'action_scheduler_failed_execution', 'my_plugin_log_failed_action', 10, 2 );
function my_plugin_log_failed_action( $action_id, $exception ) {
	$action = ActionScheduler::store()->fetch_action( $action_id );
	if ( 'my-plugin' !== $action->get_group() ) {
		return;
	}
	wc_get_logger()->error(
		sprintf( 'Action %d (%s) failed: %s', $action_id, $action->get_hook(), $exception->getMessage() ),
		array( 'source' => 'my-plugin' )
	);
}
```

## What WooCommerce schedules itself

Group `woocommerce`, unique, registered on `action_scheduler_ensure_recurring_actions`: `woocommerce_scheduled_sales` (daily at local midnight), `woocommerce_cancel_unpaid_orders` (single, re-armed every hold-stock interval), `woocommerce_cleanup_personal_data`, `woocommerce_cleanup_logs`, `woocommerce_cleanup_sessions` (12 h), `woocommerce_geoip_updater` (15 days), `woocommerce_cleanup_rate_limits_wrapper`, `wc_admin_daily_wrapper`. Also: webhook delivery `woocommerce_deliver_webhook_async` (group `woocommerce-webhooks`), DB updates `woocommerce_run_update_callback` (`woocommerce-db-updates`), attribute lookup regeneration, HPOS background sync, deferred transactional emails and analytics imports. Don't cancel or reschedule these.

## `WC()->queue()` vs `as_*`

`WC()->queue()` (`WC_Action_Queue`: `add`, `schedule_single`, `schedule_recurring`, `schedule_cron`, `cancel`, `cancel_all`, `get_next`, `search`) wraps AS **without `$unique` or `$priority`**. Use the `as_*` functions directly.

## Example A: ERP stock sync every 15 minutes, in batches

About 300 stock-managed SKUs, 50 per batch, updated through CRUD. The ERP endpoint (`POST /stock/lookup` → `{ "stock": { "SKU": qty } }`) is illustrative: adapt it to your ERP.

```php
<?php
/**
 * Plugin Name:       My Plugin ERP Stock Sync
 * Requires at least: 7.0
 * Requires PHP:      8.4
 * Requires Plugins:  woocommerce
 * Text Domain:       my-plugin
 *
 * Needs MY_PLUGIN_ERP_URL and MY_PLUGIN_ERP_TOKEN defined in wp-config.php.
 */

defined( 'ABSPATH' ) || exit;

const MY_PLUGIN_GROUP           = 'my-plugin';
const MY_PLUGIN_STOCK_TICK      = 'my_plugin_stock_sync_tick';
const MY_PLUGIN_STOCK_BATCH     = 'my_plugin_stock_sync_batch';
const MY_PLUGIN_STOCK_PER_BATCH = 50;

add_action( 'before_woocommerce_init', static function () {
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
} );

register_activation_hook( __FILE__, 'my_plugin_stock_register_schedule' );
add_action( 'action_scheduler_ensure_recurring_actions', 'my_plugin_stock_register_schedule' );
function my_plugin_stock_register_schedule() {
	if ( function_exists( 'as_schedule_recurring_action' ) ) {
		as_schedule_recurring_action( time() + MINUTE_IN_SECONDS, 15 * MINUTE_IN_SECONDS, MY_PLUGIN_STOCK_TICK, array(), MY_PLUGIN_GROUP, true );
	}
}

register_deactivation_hook( __FILE__, 'my_plugin_stock_unschedule' );
function my_plugin_stock_unschedule() {
	if ( function_exists( 'as_unschedule_all_actions' ) ) {
		as_unschedule_all_actions( '', array(), MY_PLUGIN_GROUP );
	}
}

// Tick: start a run unless the previous one is still going.
add_action( MY_PLUGIN_STOCK_TICK, 'my_plugin_stock_sync_tick' );
function my_plugin_stock_sync_tick() {
	if ( ! defined( 'MY_PLUGIN_ERP_URL' ) || ! defined( 'MY_PLUGIN_ERP_TOKEN' ) ) {
		wc_get_logger()->warning( 'Stock sync skipped: ERP credentials are not configured.', array( 'source' => 'my-plugin' ) );
		return;
	}
	if ( as_has_scheduled_action( MY_PLUGIN_STOCK_BATCH, null, MY_PLUGIN_GROUP ) ) {
		return;
	}
	as_enqueue_async_action( MY_PLUGIN_STOCK_BATCH, array( 1 ), MY_PLUGIN_GROUP, true );
}

// Batch: one page of stock-managed products and variations.
add_action( MY_PLUGIN_STOCK_BATCH, 'my_plugin_stock_sync_batch', 10, 1 );
function my_plugin_stock_sync_batch( $page ) {
	$page = max( 1, (int) $page );
	$ids  = wc_get_products(
		array(
			'type'         => array_merge( array_keys( wc_get_product_types() ), array( 'variation' ) ),
			'status'       => array( 'publish', 'private' ),
			'manage_stock' => true,
			'orderby'      => 'ID',
			'order'        => 'ASC',
			'limit'        => MY_PLUGIN_STOCK_PER_BATCH,
			'page'         => $page,
			'return'       => 'ids',
		)
	);
	if ( ! $ids ) {
		return;
	}

	$by_sku = array();
	foreach ( $ids as $id ) {
		$product = wc_get_product( $id );
		if ( $product && '' !== $product->get_sku() ) {
			$by_sku[ $product->get_sku() ] = $product;
		}
	}

	if ( $by_sku ) {
		// Throws on failure: the action is marked failed and the next tick starts over.
		$levels  = my_plugin_erp_fetch_stock( array_keys( $by_sku ) );
		$updated = 0;
		foreach ( $levels as $sku => $quantity ) {
			$sku = (string) $sku;
			if ( ! isset( $by_sku[ $sku ] ) || ! is_numeric( $quantity ) ) {
				continue;
			}
			$product  = $by_sku[ $sku ];
			$quantity = (int) $quantity;
			if ( (int) $product->get_stock_quantity() !== $quantity ) {
				wc_update_product_stock( $product, $quantity, 'set' ); // CRUD: status, lookups, caches, hooks.
				++$updated;
			}
		}
		if ( $updated ) {
			wc_get_logger()->info( sprintf( 'Stock sync page %d: %d product(s) updated.', $page, $updated ), array( 'source' => 'my-plugin' ) );
		}
	}

	if ( count( $ids ) === MY_PLUGIN_STOCK_PER_BATCH ) {
		as_enqueue_async_action( MY_PLUGIN_STOCK_BATCH, array( $page + 1 ), MY_PLUGIN_GROUP, true );
	}
}

/**
 * @param string[] $skus
 * @return array<string,int|string> SKU => quantity.
 * @throws RuntimeException On transport, HTTP or payload errors.
 */
function my_plugin_erp_fetch_stock( array $skus ) {
	$response = wp_remote_post(
		MY_PLUGIN_ERP_URL . '/stock/lookup',
		array(
			'timeout' => 15,
			'headers' => array(
				'Authorization' => 'Bearer ' . MY_PLUGIN_ERP_TOKEN,
				'Content-Type'  => 'application/json',
				'Accept'        => 'application/json',
			),
			'body'    => wp_json_encode( array( 'skus' => array_values( $skus ) ) ),
		)
	);
	if ( is_wp_error( $response ) ) {
		throw new RuntimeException( 'ERP stock lookup failed: ' . $response->get_error_message() );
	}
	$code = (int) wp_remote_retrieve_response_code( $response );
	if ( 200 !== $code ) {
		throw new RuntimeException( sprintf( 'ERP stock lookup returned HTTP %d.', $code ) );
	}
	$data = json_decode( wp_remote_retrieve_body( $response ), true );
	if ( ! is_array( $data ) || ! isset( $data['stock'] ) || ! is_array( $data['stock'] ) ) {
		throw new RuntimeException( 'ERP stock lookup returned an unexpected payload.' );
	}
	return $data['stock'];
}
```

Why it looks like this: the recurring tick is tiny and unique; the starter debounces overlapping runs; each batch makes one ERP call for 50 SKUs and stays far under the 30 s run limit; stock changes go through `wc_update_product_stock()` and only when the value differs. A failed batch isn't retried: the next tick, 15 minutes later, is the retry.

## Example B: push each paid order to an ERP, with retries and backoff

The status hook only enqueues. The job is idempotent (stored ERP reference plus an idempotency key), retries transient errors with exponential backoff, and throws on the final failure.

```php
<?php
/**
 * Plugin Name:       My Plugin ERP Orders
 * Requires at least: 7.0
 * Requires PHP:      8.4
 * Requires Plugins:  woocommerce
 * Text Domain:       my-plugin
 *
 * Needs MY_PLUGIN_ERP_URL and MY_PLUGIN_ERP_TOKEN defined in wp-config.php.
 */

defined( 'ABSPATH' ) || exit;

const MY_PLUGIN_ERP_GROUP        = 'my-plugin';
const MY_PLUGIN_ERP_PUSH_HOOK    = 'my_plugin_erp_push_order';
const MY_PLUGIN_ERP_MAX_ATTEMPTS = 6;
const MY_PLUGIN_ERP_REF_META     = '_my_plugin_erp_ref';

add_action( 'before_woocommerce_init', static function () {
	\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility( 'custom_order_tables', __FILE__, true );
} );

// 1. Enqueue only. Runs inside the Store API request, the gateway webhook or the admin save.
add_action( 'woocommerce_order_status_changed', 'my_plugin_queue_erp_push', 10, 4 );
function my_plugin_queue_erp_push( $order_id, $from, $to, $order ) {
	if ( ! $order instanceof WC_Order || ! in_array( $to, wc_get_is_paid_statuses(), true ) ) {
		return;
	}
	if ( '' !== (string) $order->get_meta( MY_PLUGIN_ERP_REF_META ) ) {
		return; // Already in the ERP (e.g. processing -> completed).
	}
	as_enqueue_async_action( MY_PLUGIN_ERP_PUSH_HOOK, array( (int) $order_id, 1 ), MY_PLUGIN_ERP_GROUP, true );
}

// 2. The job. Args arrive positionally: ( $order_id, $attempt ).
add_action( MY_PLUGIN_ERP_PUSH_HOOK, 'my_plugin_erp_push_order', 10, 2 );
function my_plugin_erp_push_order( $order_id, $attempt = 1 ) {
	$order_id = (int) $order_id;
	$attempt  = max( 1, (int) $attempt );
	$order    = wc_get_order( $order_id );

	if ( ! $order || '' !== (string) $order->get_meta( MY_PLUGIN_ERP_REF_META ) ) {
		return; // Deleted, or a parallel attempt already succeeded.
	}

	$result = my_plugin_erp_send_order( $order );

	if ( $result['ok'] ) {
		$order->update_meta_data( MY_PLUGIN_ERP_REF_META, $result['ref'] );
		$order->save();
		/* translators: %s: ERP reference. */
		$order->add_order_note( sprintf( __( 'Order sent to the ERP (reference %s).', 'my-plugin' ), $result['ref'] ) );
		return;
	}

	$logger = wc_get_logger();

	if ( $result['retry'] && $attempt < MY_PLUGIN_ERP_MAX_ATTEMPTS ) {
		$delay = (int) min( HOUR_IN_SECONDS, MINUTE_IN_SECONDS * ( 2 ** ( $attempt - 1 ) ) ); // 1, 2, 4, 8, 16 min.
		// New args ( attempt + 1 ) differ from the running action, so $unique doesn't block the insert.
		as_schedule_single_action( time() + $delay, MY_PLUGIN_ERP_PUSH_HOOK, array( $order_id, $attempt + 1 ), MY_PLUGIN_ERP_GROUP, true );
		$logger->warning(
			sprintf( 'ERP push for order %d failed (attempt %d): %s. Retrying in %d s.', $order_id, $attempt, $result['message'], $delay ),
			array( 'source' => 'my-plugin' )
		);
		return; // This attempt completes; the retry is pending.
	}

	/* translators: %s: error message. */
	$order->add_order_note( sprintf( __( 'Could not send the order to the ERP: %s', 'my-plugin' ), $result['message'] ) );
	// Final failure: throw so the action is marked failed and shows up in --status=failed.
	throw new RuntimeException( sprintf( 'ERP push for order %d failed after %d attempt(s): %s', $order_id, $attempt, $result['message'] ) );
}

/**
 * @return array{ok: bool, retry: bool, ref: string, message: string}
 */
function my_plugin_erp_send_order( WC_Order $order ) {
	if ( ! defined( 'MY_PLUGIN_ERP_URL' ) || ! defined( 'MY_PLUGIN_ERP_TOKEN' ) ) {
		return array( 'ok' => false, 'retry' => false, 'ref' => '', 'message' => 'ERP credentials are not configured' );
	}

	$decimals = wc_get_price_decimals();
	$lines    = array();
	foreach ( $order->get_items() as $item ) {
		if ( ! $item instanceof WC_Order_Item_Product ) {
			continue;
		}
		$product = $item->get_product();
		$lines[] = array(
			'sku'      => $product ? $product->get_sku() : '',
			'quantity' => (int) $item->get_quantity(),
			'total'    => wc_format_decimal( $item->get_total(), $decimals ), // String, no float math.
		);
	}

	$response = wp_remote_post(
		MY_PLUGIN_ERP_URL . '/orders',
		array(
			'timeout' => 15,
			'headers' => array(
				'Authorization'   => 'Bearer ' . MY_PLUGIN_ERP_TOKEN,
				'Content-Type'    => 'application/json',
				'Accept'          => 'application/json',
				'Idempotency-Key' => 'wc-order-' . $order->get_id(), // If the ERP supports it.
			),
			'body'    => wp_json_encode(
				array(
					'external_id' => $order->get_id(),
					'number'      => $order->get_order_number(),
					'currency'    => $order->get_currency(),
					'total'       => wc_format_decimal( $order->get_total(), $decimals ),
					'email'       => $order->get_billing_email(),
					'lines'       => $lines,
				)
			),
		)
	);

	if ( is_wp_error( $response ) ) {
		return array( 'ok' => false, 'retry' => true, 'ref' => '', 'message' => sanitize_text_field( $response->get_error_message() ) );
	}

	$code = (int) wp_remote_retrieve_response_code( $response );
	$body = json_decode( wp_remote_retrieve_body( $response ), true );

	if ( $code >= 200 && $code < 300 && is_array( $body ) && ! empty( $body['id'] ) ) {
		return array( 'ok' => true, 'retry' => false, 'ref' => sanitize_text_field( (string) $body['id'] ), 'message' => '' );
	}

	// 408, 429 and 5xx are transient; any other answer means payload or credentials need a human.
	$retry = 408 === $code || 429 === $code || $code >= 500;

	return array( 'ok' => false, 'retry' => $retry, 'ref' => '', 'message' => sprintf( 'HTTP %d', $code ) );
}
```

Race notes: a second paid transition while attempt 1 is pending is deduplicated by `unique`. One that arrives while a retry (`attempt` ≥ 2) is pending enqueues a parallel attempt 1. Both check the stored reference first, and the idempotency key stops the ERP from creating a duplicate.

## Common mistakes

- `wp_schedule_event()` + `cron_schedules` for plugin jobs. Use `as_schedule_recurring_action()` / `as_schedule_cron_action()`.
- Assuming AS runs without WP-Cron or a worker. On a fully cached front end with `DISABLE_WP_CRON` and no system cron, nothing runs.
- Calling `as_*` at plugin load or before `init` priority 1. You get `0` and a `_doing_it_wrong`.
- Registering a recurring action only on `action_scheduler_ensure_recurring_actions`, so it first appears tomorrow at 3am. Also schedule on activation.
- Re-checking `as_has_scheduled_action()` + scheduling on every request instead of `unique = true` on the ensure hook.
- Rescheduling from inside the callback with the **same** args and `unique = true`. The running action blocks the insert; add an attempt counter.
- Expecting failed actions to retry. They don't, and a recurring action stops after 5 consecutive failures.
- Throwing on a transient error you meant to retry (no retry happens), or swallowing the final failure (nothing shows as failed).
- Passing objects, arrays of order data or prices as args. Pass ids, cast them, and re-read inside the job.
- Writing the callback as `function ( $args )`, expecting the array. Args are spread positionally, and `accepted_args` must match.
- Registering job callbacks only in admin or only on the front end, so the CLI worker fails them with "no callbacks are registered".
- Calling the ERP/CRM inline in `woocommerce_payment_complete` or `woocommerce_order_status_*`.
- One giant action that loops over every product. Batch with a cursor and keep each action under 30 s.
- Page cursors over a set the job itself shrinks ("unexported" → exported), which skips items.
- Using `WC()->queue()` when you need uniqueness or priority.
- Leaving actions behind on deactivation. `as_unschedule_all_actions( '', [], 'my-plugin' )`.
- Forgetting that a stock change can flip stock status on cached product pages. If you run an edge cache, enqueue a debounced purge for the affected URLs.
