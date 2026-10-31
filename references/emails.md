# Transactional emails

How to add and send WooCommerce emails in 11.2.1: the `WC_Email` contract, where to load the class, how status transitions reach an email, a complete custom-status email with a tracking link, the block email editor, `email_improvements`, deferred sending through Action Scheduler, the email log hooks and admin previews.

## `WC_Email` contract (11.2.1)

`includes/emails/class-wc-email.php`, extends `WC_Settings_API`.

| Property | Notes |
|---|---|
| `$id` | Unique id; settings live in `woocommerce_{id}_settings`. |
| `$title`, `$description` | Admin list and settings screen. |
| `$customer_email` (protected) | `true` = recipient is set in code (no recipient field). |
| `$manual` (protected) | `true` for emails only sent on demand. |
| `$email_group` | Groups the emails list: `accounts`, `orders`, `order-updates`, `order-changes`, `payments` (`order-processing` and `order-exceptions` are deprecated). Filter `woocommerce_email_groups` (10.3). |
| `$template_html`, `$template_plain` | Paths relative to `$template_base`, overridable from the theme (`yourtheme/woocommerce/…`). |
| `$template_block` | Initial content for the block email editor. If empty, derived from `$template_plain` by replacing `plain` with `block`. |
| `$template_base` | Your plugin's `templates/` dir. Defaults to WooCommerce's when null. |
| `$placeholders` | Set in the child constructor **before** `parent::__construct()`; the parent merges in `{site_title}`, `{site_address}`, `{site_url}`, `{store_email}`. Used in subject, heading and additional content. |
| `$object` | Usually the `WC_Order`. |
| `$recipient`, `$cc`, `$bcc` | Cc/Bcc fields exist only with `email_improvements` on and `$supports_cc_bcc` (protected, 11.2, default `true`). |
| `$email_improvements_enabled`, `$block_email_editor_enabled` | Feature flags, set by the parent constructor. |
| `$template_block_content` | Template that fills the editor's `woocommerce/email-content` block. Default `emails/block/general-block-email.php`. |

| Method | Notes |
|---|---|
| `get_default_subject()`, `get_default_heading()`, `get_default_additional_content()` | Override; the settings fields default to empty and fall back to these. |
| `get_content_html()`, `get_content_plain()` | **Return `''` in the base class.** Override both. |
| `get_content()` | Block editor content first (when enabled and opted in), else plain or HTML by `email_type`. |
| `setup_locale()` / `restore_locale()` | Wrap `trigger()` so customer emails use the right language. |
| `send_notification()` | Protected, 10.9. Checks enabled + recipient, fires `woocommerce_email_disabled` / `woocommerce_email_skipped`, then `send()`. Use it from `trigger()`. |
| `send_if_recipient()` | Protected, 10.9. For manual emails that ignore the enabled setting. |
| `send( $to, $subject, $message, $headers, $attachments )` | Fires `woocommerce_email_sent( $ok, $id, $email )`. Since 11.1 a `woocommerce_mail_callback` that does not return a bool triggers `_doing_it_wrong`. |
| `get_preheader()` | Preheader field (only with the block email editor on). |
| `const SKIP_REASON_NO_RECIPIENT` | `'no_recipient'`, 10.9. |

Default settings fields: `enabled` (default `yes`), `subject`, `heading`, `additional_content`, `email_type` (default `html`), plus `cc`/`bcc` and `preheader` when their features are on.

## Loading the class

`WC_Email` is included by `WC_Emails::init()` immediately before it applies `woocommerce_email_classes`. Defining `class X extends WC_Email` at plugin load fatals (or depends on load order). Require the file **inside** the filter callback:

```php
add_filter(
	'woocommerce_email_classes',
	static function ( array $emails ) {
		require_once __DIR__ . '/includes/class-my-plugin-email-order-shipped.php';
		$emails['My_Plugin_Email_Order_Shipped'] = new My_Plugin_Email_Order_Shipped();
		return $emails;
	}
);
```

The array key is the class name; the admin preview uses it as the email type.

## How a status change reaches your email

```
$order->update_status( 'shipped' )  ─► save() ─► status_transition()
   do_action( 'woocommerce_order_status_shipped', $id, $order, $transition )
   do_action( 'woocommerce_order_status_processing_to_shipped', $id, $order )
   do_action( 'woocommerce_order_status_changed', $id, $from, $to, $order )

WC_Emails::init_transactional_emails()  (only for hooks listed in woocommerce_email_actions)
   deferred off ─► send_transactional_email ─► do_action( '{hook}_notification', ...args )   same request
   deferred on  ─► queue_transactional_email ─► one Action Scheduler action per email on shutdown
                   (hook woocommerce_send_queued_transactional_email, group woocommerce-emails)
                   ─► later: do_action( '{hook}_notification', ...args re-fetched by id )

Your email: add_action( 'woocommerce_order_status_shipped_notification', [ $this, 'trigger' ], 10, 2 )
```

The `_notification` variants exist **only** for hooks in the `woocommerce_email_actions` list. Core lists transitions such as `woocommerce_order_status_pending_to_processing` and `woocommerce_order_status_completed`; a custom status is not there until you add it. Listening to `woocommerce_order_status_shipped` directly works but skips deferral and runs your rendering inside the admin's save request.

## Complete example: "Despachado" (shipped) email with tracking

The `wc-shipped` status itself (registered in both `woocommerce_register_shop_order_post_statuses` and `wc_order_statuses`, slug ≤ 20 chars with `wc-`) is covered in `orders-hpos.md`. Source strings are English; the `es_CL` translation supplies "Despachado".

```
my-plugin/
├── my-plugin.php
├── includes/class-my-plugin-email-order-shipped.php
└── templates/emails/
    ├── my-plugin-order-shipped.php
    ├── plain/my-plugin-order-shipped.php
    └── block/my-plugin-order-shipped.php
```

`my-plugin.php` (email part):

```php
<?php
defined( 'ABSPATH' ) || exit;

define( 'MY_PLUGIN_FILE', __FILE__ );

// 1. Make the transition email-capable (needed for *_notification and for deferred sending).
add_filter(
	'woocommerce_email_actions',
	static function ( array $actions ) {
		$actions[] = 'woocommerce_order_status_shipped';
		return $actions;
	}
);

// 2. Register the email.
add_filter(
	'woocommerce_email_classes',
	static function ( array $emails ) {
		require_once __DIR__ . '/includes/class-my-plugin-email-order-shipped.php';
		$emails['My_Plugin_Email_Order_Shipped'] = new My_Plugin_Email_Order_Shipped();
		return $emails;
	}
);

// 3. Opt into the block email editor (no effect while the editor feature is off).
add_filter(
	'woocommerce_transactional_emails_for_block_editor',
	static function ( array $ids ) {
		$ids[] = 'my_plugin_order_shipped';
		return $ids;
	}
);

/**
 * Persist everything the email needs BEFORE the status change: a deferred email
 * re-fetches the order in a later request and sees only saved data.
 */
function my_plugin_mark_order_shipped( WC_Order $order, string $tracking_url, string $tracking_number ): void {
	$order->update_meta_data( '_my_plugin_tracking_url', esc_url_raw( $tracking_url ) );
	$order->update_meta_data( '_my_plugin_tracking_number', sanitize_text_field( $tracking_number ) );
	$order->update_status( 'shipped', __( 'Order shipped.', 'my-plugin' ) ); // One save(): meta + status, then hooks.
}
```

`includes/class-my-plugin-email-order-shipped.php`:

```php
<?php
defined( 'ABSPATH' ) || exit;

class My_Plugin_Email_Order_Shipped extends WC_Email {

	public function __construct() {
		$this->id             = 'my_plugin_order_shipped';
		$this->customer_email = true;
		$this->title          = __( 'Order shipped', 'my-plugin' );
		$this->description    = __( 'Sent to the customer when the order is marked as shipped, with the tracking link.', 'my-plugin' );
		$this->email_group    = 'order-updates';
		$this->template_html  = 'emails/my-plugin-order-shipped.php';
		$this->template_plain = 'emails/plain/my-plugin-order-shipped.php';
		$this->template_block = 'emails/block/my-plugin-order-shipped.php';
		$this->template_base  = plugin_dir_path( MY_PLUGIN_FILE ) . 'templates/';
		$this->placeholders   = array(
			'{order_date}'   => '',
			'{order_number}' => '',
		);

		add_action( 'woocommerce_order_status_shipped_notification', array( $this, 'trigger' ), 10, 2 );

		parent::__construct();
	}

	public function get_default_subject() {
		return __( 'Your {site_title} order #{order_number} is on its way', 'my-plugin' );
	}

	public function get_default_heading() {
		return __( 'Your order is on its way', 'my-plugin' );
	}

	public function get_default_additional_content() {
		return __( 'If you have any questions, write to us at {store_email}.', 'my-plugin' );
	}

	/**
	 * @param int           $order_id Order id.
	 * @param WC_Order|bool $order    Order object (re-fetched by id when sent from Action Scheduler).
	 */
	public function trigger( $order_id, $order = false ) {
		$this->setup_locale();

		if ( $order_id && ! $order instanceof WC_Order ) {
			$order = wc_get_order( $order_id );
		}
		if ( $order instanceof WC_Order ) {
			$this->object                         = $order;
			$this->recipient                      = $order->get_billing_email();
			$this->placeholders['{order_date}']   = wc_format_datetime( $order->get_date_created() );
			$this->placeholders['{order_number}'] = $order->get_order_number();
		}

		$this->send_notification(); // Enabled + recipient checks, log events, send().

		$this->restore_locale();
	}

	public function get_content_html() {
		return wc_get_template_html( $this->template_html, $this->get_template_args( false ), '', $this->template_base );
	}

	public function get_content_plain() {
		return wc_get_template_html( $this->template_plain, $this->get_template_args( true ), '', $this->template_base );
	}

	private function get_template_args( bool $plain_text ): array {
		$order = $this->object instanceof WC_Order ? $this->object : null;
		return array(
			'order'              => $order,
			'email_heading'      => $this->get_heading(),
			'additional_content' => $this->get_additional_content(),
			'sent_to_admin'      => false,
			'plain_text'         => $plain_text,
			'email'              => $this,
			'tracking_url'       => $order ? (string) $order->get_meta( '_my_plugin_tracking_url' ) : '',
			'tracking_number'    => $order ? (string) $order->get_meta( '_my_plugin_tracking_number' ) : '',
		);
	}
}
```

`templates/emails/my-plugin-order-shipped.php`:

```php
<?php
/**
 * Order shipped (HTML). Override: yourtheme/woocommerce/emails/my-plugin-order-shipped.php
 *
 * @var WC_Order $order
 * @var string   $email_heading
 * @var string   $additional_content
 * @var bool     $sent_to_admin
 * @var bool     $plain_text
 * @var WC_Email $email
 * @var string   $tracking_url
 * @var string   $tracking_number
 */

defined( 'ABSPATH' ) || exit;

do_action( 'woocommerce_email_header', $email_heading, $email ); ?>

<p>
<?php
/* translators: %s: customer first name */
printf( esc_html__( 'Hi %s,', 'my-plugin' ), esc_html( $order->get_billing_first_name() ) );
?>
</p>
<p><?php esc_html_e( 'Good news: your order has left our warehouse.', 'my-plugin' ); ?></p>

<?php if ( $tracking_url ) : ?>
	<p>
		<a href="<?php echo esc_url( $tracking_url ); ?>"><?php esc_html_e( 'Track your order', 'my-plugin' ); ?></a>
		<?php if ( $tracking_number ) : ?>
			<?php /* translators: %s: tracking number */ ?>
			(<?php printf( esc_html__( 'tracking number %s', 'my-plugin' ), esc_html( $tracking_number ) ); ?>)
		<?php endif; ?>
	</p>
<?php endif; ?>

<?php
do_action( 'woocommerce_email_order_details', $order, $sent_to_admin, $plain_text, $email );
do_action( 'woocommerce_email_order_meta', $order, $sent_to_admin, $plain_text, $email );
do_action( 'woocommerce_email_customer_details', $order, $sent_to_admin, $plain_text, $email );

if ( $additional_content ) {
	echo wp_kses_post( wpautop( wptexturize( $additional_content ) ) );
}

do_action( 'woocommerce_email_footer', $email );
```

`templates/emails/plain/my-plugin-order-shipped.php`:

```php
<?php
defined( 'ABSPATH' ) || exit;

echo "=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=\n";
echo esc_html( wp_strip_all_tags( $email_heading ) );
echo "\n=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=\n\n";

/* translators: %s: customer first name */
echo sprintf( esc_html__( 'Hi %s,', 'my-plugin' ), esc_html( $order->get_billing_first_name() ) ) . "\n\n";
echo esc_html__( 'Good news: your order has left our warehouse.', 'my-plugin' ) . "\n\n";
if ( $tracking_url ) {
	echo esc_html__( 'Track your order:', 'my-plugin' ) . ' ' . esc_url_raw( $tracking_url ) . "\n\n";
}

do_action( 'woocommerce_email_order_details', $order, $sent_to_admin, $plain_text, $email );
echo "\n----------------------------------------\n\n";
do_action( 'woocommerce_email_customer_details', $order, $sent_to_admin, $plain_text, $email );

if ( $additional_content ) {
	echo "\n" . esc_html( wp_strip_all_tags( wptexturize( $additional_content ) ) ) . "\n";
}

echo wp_kses_post( apply_filters( 'woocommerce_email_footer_text', get_option( 'woocommerce_email_footer_text' ) ) );
```

`templates/emails/block/my-plugin-order-shipped.php` and the tracking button for the block editor are in the next section.

## Block email editor

Status in 11.2.1: **alpha, off by default** ("Block Email Editor (alpha)", `enabled_by_default` false, experimental). The package activates only when option `woocommerce_feature_block_email_editor_enabled` is `yes`. Write the classic `get_content_html/plain` path first; the block path is an addition.

How it works:

- Emails are posts of type `woo_email`, created lazily as **drafts** when the merchant opens the editor. Until a post is published, your file template (`$template_block`) is what renders.
- Third-party emails must opt in with `woocommerce_transactional_emails_for_block_editor` (9.9). Without it, block rendering returns null and the classic templates are used.
- The block template is located with `wc_locate_template( $template_block, '', $template_base )` (theme-overridable); a missing file falls back to `emails/block/default-block-content.php`. Filters: `woocommerce_email_block_template_html( $html, $email )` (10.7, also on the send path) and `woocommerce_email_content_post_data( $post_data, $type, $email )` (10.5; `post_status` is ignored).
- The template must contain the locked `woocommerce/email-content` block holding the placeholder `##WOO_CONTENT##` (value of the internal constant `BlockEmailRenderer::WOO_EMAIL_CONTENT_PLACEHOLDER`). It is replaced with `$template_block_content` (default `emails/block/general-block-email.php`) rendered with `order`, `sent_to_admin`, `plain_text`, `email`.
- The general template fires **`woocommerce_email_general_block_content( $sent_to_admin, $plain_text, $email )`** (10.5), then order details, order meta and customer details, unless the email id is in `woocommerce_emails_general_block_content_emails_without_order_details`. The developer docs call this action `woocommerce_email_general_block_email`: that name does not exist in 11.2.1.
- Personalization tags such as `<!--[woocommerce/customer-first-name]-->`, `<!--[woocommerce/order-number]-->`, `<!--[woocommerce/store-email]-->` are resolved at send time.
- With the editor on, core retitles some emails (Completed order becomes "Order fulfilled") and adds a preheader field.

`templates/emails/block/my-plugin-order-shipped.php`:

```php
<?php
/**
 * Order shipped: initial block email content.
 */

defined( 'ABSPATH' ) || exit;

// phpcs:disable Squiz.PHP.EmbeddedPhp.ContentBeforeOpen, Squiz.PHP.EmbeddedPhp.ContentAfterEnd
?>

<!-- wp:heading -->
<h2 class="wp-block-heading"><?php echo esc_html__( 'Your order is on its way', 'my-plugin' ); ?></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p><?php
	/* translators: %s: customer first name personalization tag */
	printf( esc_html__( 'Hi %s,', 'my-plugin' ), '<!--[woocommerce/customer-first-name]-->' );
?></p>
<!-- /wp:paragraph -->

<!-- wp:woocommerce/email-content {"lock":{"move":false,"remove":true}} -->
<div class="wp-block-woocommerce-email-content">##WOO_CONTENT##</div>
<!-- /wp:woocommerce/email-content -->
```

The tracking button goes inside the content block through the real action (`emails/email-button.php` ships since 11.0):

```php
add_action(
	'woocommerce_email_general_block_content',
	static function ( $sent_to_admin, $plain_text, $email ) {
		if ( ! $email instanceof WC_Email || 'my_plugin_order_shipped' !== $email->id || ! $email->object instanceof WC_Order ) {
			return;
		}
		$url = (string) $email->object->get_meta( '_my_plugin_tracking_url' );
		if ( '' !== $url ) {
			wc_get_template(
				'emails/email-button.php',
				array(
					'url'   => $url,
					'label' => __( 'Track your order', 'my-plugin' ),
				)
			);
		}
	},
	10,
	3
);
```

Optional: a personalization tag the merchant can insert as text in the editor.

```php
use Automattic\WooCommerce\EmailEditor\Engine\PersonalizationTags\Personalization_Tag;

add_filter(
	'woocommerce_email_editor_register_personalization_tags',
	static function ( $registry ) {
		$registry->register(
			new Personalization_Tag(
				__( 'Tracking number', 'my-plugin' ),
				'my-plugin/tracking-number', // Token in templates: <!--[my-plugin/tracking-number]-->.
				__( 'Order', 'my-plugin' ),
				static function ( array $context ): string {
					$order = isset( $context['order'] ) ? $context['order'] : null; // WC_Order or null.
					return $order instanceof WC_Order ? esc_html( (string) $order->get_meta( '_my_plugin_tracking_number' ) ) : '';
				},
				array(),
				null,
				array( 'woo_email' )
			)
		);
		return $registry;
	}
);
```

The callback context holds `order`, `wp_user`, `wc_email`, `recipient_email`; add more with `woocommerce_email_editor_integration_personalizer_context_data( $context, $email )` (10.5).

## `email_improvements`

Not experimental. Enabled automatically on **new installs**; the rollout to existing stores is currently paused. When on: modern templates and copy (core templates branch on `FeaturesUtil::feature_is_enabled( 'email_improvements' )`), Cc/Bcc fields, the new email settings and preview UI. In your class, read `$this->email_improvements_enabled` **after** `parent::__construct()` if you want copy parity; set `protected $supports_cc_bcc = false;` (11.2) to hide Cc/Bcc for an email.

## Deferred emails (Action Scheduler)

- Feature "Deferred emails" (`deferred_transactional_emails`), off by default, not experimental. Filter `woocommerce_defer_transactional_emails` defaults to the feature flag.
- `DeferredEmailQueue` (10.8) collects every `woocommerce_email_actions` hook fired during the request and, on `shutdown`, enqueues one action per email (`woocommerce_send_queued_transactional_email`, group `woocommerce-emails`).
- Arguments are stored in Action Scheduler: scalars and arrays as-is; `WC_Order`, `WC_Product`, `WC_Payment_Gateway` (and core stock notifications) by id, **re-fetched at send time**. Any other object makes that email fall back to synchronous sending.
- Consequences: trigger arguments must be ids or those WC objects, and anything the email shows must be saved before the hook fires (as `my_plugin_mark_order_shipped()` does). Nothing held in memory or in a global survives.
- Gate a queued email with `woocommerce_allow_send_queued_transactional_email( true, $filter, $args )`.

Delayed emails follow core's review-request pattern: schedule an action, list it in `woocommerce_email_actions`, listen on its `_notification`.

```php
// When the order ships, schedule a follow-up in 7 days.
add_action(
	'woocommerce_order_status_shipped',
	static function ( $order_id ) {
		as_schedule_single_action( time() + 7 * DAY_IN_SECONDS, 'my_plugin_delivery_followup', array( (int) $order_id ), 'my-plugin', true );
	}
);
add_filter(
	'woocommerce_email_actions',
	static function ( array $actions ) {
		$actions[] = 'my_plugin_delivery_followup'; // Your email listens on my_plugin_delivery_followup_notification.
		return $actions;
	}
);
```

## Email log

`EmailLogger` (10.9, internal, always on) writes every attempt to the WooCommerce logs under source **`transactional-emails`** (email type, related object, recipient as username or `guest`, outcome, `wp_mail_failed` reason). Public hooks if you need your own reporting:

| Hook | Args |
|---|---|
| `woocommerce_email_sent` | `bool $success, string $email_id, WC_Email $email` |
| `woocommerce_email_disabled` | `string $email_id, WC_Email $email` (fired by `send_notification()`) |
| `woocommerce_email_skipped` | `string $reason, string $email_id, WC_Email $email` (`WC_Email::SKIP_REASON_NO_RECIPIENT`) |

Calling `send()` directly bypasses the disabled/skipped events: use `send_notification()`.

## Previews

The admin preview renders any registered email with a **dummy, unsaved order** (no id, none of your meta). Make templates tolerate empty meta and fill sample data:

```php
add_filter(
	'woocommerce_prepare_email_for_preview',
	static function ( $email ) {
		if ( 'my_plugin_order_shipped' === $email->id && $email->object instanceof WC_Order ) {
			$email->object->update_meta_data( '_my_plugin_tracking_url', 'https://tracking.example.com/ABC123' ); // In memory only.
			$email->object->update_meta_data( '_my_plugin_tracking_number', 'ABC123' );
		}
		return $email;
	}
);
```

Other preview hooks (9.6+): `woocommerce_email_preview_placeholders( $placeholders, $email_type, $object )`, `woocommerce_email_preview_dummy_order|product|product_variation|downloadable_product|address`, `woocommerce_email_preview_show_shipping_details` (11.0). `apply_filters( 'woocommerce_is_email_preview', false )` is true while a preview renders: skip side effects there.

## Common mistakes

- `class X extends WC_Email` at plugin load instead of inside `woocommerce_email_classes`.
- Hooking `woocommerce_order_status_{from}_to_{to}_notification` (or `…_{to}_notification`) without adding the base hook to `woocommerce_email_actions`: it never fires.
- `if ( $this->is_enabled() && $this->get_recipient() ) $this->send( … )`: use `send_notification()` so the log and skip events fire.
- Not overriding `get_content_html()` and `get_content_plain()`: the base class returns empty strings.
- Setting `$this->placeholders` after `parent::__construct()`: the defaults are not merged and the settings help text misses your placeholders.
- Missing `setup_locale()` / `restore_locale()` around a customer email.
- Passing a custom object, closure or unsaved data as trigger arguments: deferred sending falls back to synchronous, or the email renders without the data.
- Changing the status first and saving tracking meta afterwards: the email (deferred or not) renders without the link.
- Using `woocommerce_email_general_block_email` from the docs: the real action is `woocommerce_email_general_block_content`.
- Expecting your block template to render without `woocommerce_transactional_emails_for_block_editor`, or without the locked `woocommerce/email-content` block and `##WOO_CONTENT##`.
- Building features on the block email editor as if it were stable: it is alpha and off by default in 11.2.1.
- Unescaped output in templates; tracking URLs must go through `esc_url()` (HTML) or `esc_url_raw()` (plain).
