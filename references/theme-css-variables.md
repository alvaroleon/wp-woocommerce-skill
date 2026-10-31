# Theme contract: CSS variables and presets WooCommerce expects

WooCommerce's block templates (`templates/templates/*.html`, `templates/parts/*.html`), its patterns (`patterns/*.php`) and the stylesheets of its blocks (`assets/client/blocks/*.css`, plus `assets/css/woocommerce.css` and `woocommerce-blocktheme.css`) style themselves with two kinds of custom properties:

1. **`theme.json` preset variables** (`--wp--preset--{type}--{slug}`), referenced by slug in block attributes (`"fontSize":"small"`, `var:preset|spacing|20`) or by `var()` in CSS. If your theme does not define that slug, the variable is undefined. Then the CSS fallback applies if there is one; otherwise the property inherits or is dropped.
2. **WooCommerce's own variables** (`--wc-*`), with defaults on `:root` or scoped to a block. A theme can override them.

Use this when building a block theme from scratch so the Cart, Checkout, Mini-Cart, product blocks, filters and WooCommerce patterns pick up the brand instead of falling back to hard-coded values. Inventory verified against WooCommerce 11.2.1. Re-check after major updates with the commands at the end.

## 1. Preset slugs to define in `theme.json`

"Core default" means WordPress's own `theme.json` (`wp-includes/theme.json`) already ships that slug. It keeps working only while you don't disable the default presets (`defaultPalette`, `defaultFontSizes`, `defaultSpacingSizes`). Defining the slug yourself overrides the core value, and that is what you want for brand control.

### Font sizes (`settings.typography.fontSizes`)

| Slug | Used by | Fallback in WC CSS | Core default |
|---|---|---|---|
| `small` | By far the most used: cart, checkout, Mini-Cart contents, totals, notices, grouped-product selector; also templates and patterns (`"fontSize":"small"`) | `14px` (most uses) | yes (13px) |
| `x-small` | Totals item descriptions, shipping package items, order-summary quantities (cart, checkout, Mini-Cart) | `calc(var(--wp--preset--font-size--small, 14px) * .875)` | **no** |
| `medium` | Cart/checkout headings and totals, templates, patterns, coming-soon | `22px` in some rules; **none** in coming-soon | yes (20px) |
| `large` | Templates (`"fontSize":"large"`), `woocommerce-blocktheme.css` | none | yes (36px) |
| `x-large` | Coming-soon site title (`clamp( medium, 3.5vw, x-large )`) | **none** | yes (42px) |

### Colors (`settings.color.palette`)

| Slug | Used by | Fallback | Core default |
|---|---|---|---|
| `base` | Order-summary quantity badge text (checkout), customer-account dropdown surface | `#fff` | **no** |
| `contrast` | Templates (`var(--wp--preset--color--contrast)`), patterns, product-gallery dialog close button | `#000` (gallery) / none (templates) | **no** |
| `contrast-2` | Product SKU prefix; patterns (`"textColor":"contrast-2"`) | **none** | **no** |
| `black` | "No rating" star outline | `#000` | yes |
| `luminous-vivid-orange` | Info icon on Featured Product / Featured Category | `#e26f56` | yes |
| `background`, `foreground` | `woocommerce-blocktheme.css` (older slugs, loaded only on block themes) | `#fff` / `#fcfbfe`, `#720eec` / `#958e09` | **no** |

`base` / `contrast` / `contrast-2` follow the default-theme naming (Twenty Twenty-Four onward). If your palette uses other slugs (`ink`, `paper`, `primary`…), **also** declare `base`, `contrast` and `contrast-2`, mapped to your brand colors. Otherwise those WooCommerce elements fall back to white/black or lose their color.

### Spacing (`settings.spacing.spacingSizes` or `spacingScale`)

| Slug | Used by | Fallback | Core default |
|---|---|---|---|
| `10` | Patterns (`var:preset|spacing|10`) | none | **no** (the core scale starts at `20`) |
| `20` | Templates, patterns, accordion, customer-account gutter and dropdown padding | `16px` / `1em` / `8px` depending on rule | yes |
| `30` | Product Filters overlay header/content/footer padding, customer-account dropdown, patterns | **none** in filters | yes |
| `40`, `50` | Patterns (`50` is the most used spacing in WC patterns) | none | yes |

Keep the numeric slug names (`10`–`80`). Custom names (`sm`, `md`) leave every WooCommerce reference undefined.

### Font family and layout

| Variable | Used by | Notes |
|---|---|---|
| `--wp--preset--font-family--system-sans-serif` | Product Collection preview button | No fallback, no core default; minor. Define a `system-sans-serif` family if you care. |
| `--wp--style--block-gap` | Many WC blocks (`settings.spacing.blockGap`) | Set `styles.spacing.blockGap`. |
| `--wp--style--global--wide-size` | Wide-aligned WC blocks | Set `settings.layout.wideSize` (and `contentSize`). |

### Copyable `theme.json` fragment

Values are placeholders; map them to the brand. Keep your own extra slugs alongside these.

```json
{
	"$schema": "https://schemas.wp.org/trunk/theme.json",
	"version": 3,
	"settings": {
		"layout": { "contentSize": "720px", "wideSize": "1280px" },
		"color": {
			"palette": [
				{ "slug": "base", "name": "Base", "color": "#ffffff" },
				{ "slug": "contrast", "name": "Contrast", "color": "#111111" },
				{ "slug": "contrast-2", "name": "Contrast 2", "color": "#5f5f5f" },
				{ "slug": "black", "name": "Black", "color": "#000000" }
			]
		},
		"typography": {
			"fontSizes": [
				{ "slug": "x-small", "name": "Extra small", "size": "0.75rem" },
				{ "slug": "small", "name": "Small", "size": "0.875rem" },
				{ "slug": "medium", "name": "Medium", "size": "1.125rem" },
				{ "slug": "large", "name": "Large", "size": "1.75rem" },
				{ "slug": "x-large", "name": "Extra large", "size": "2.5rem" }
			],
			"fontFamilies": [
				{ "slug": "system-sans-serif", "name": "System sans-serif", "fontFamily": "-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif" }
			]
		},
		"spacing": {
			"spacingSizes": [
				{ "slug": "10", "name": "1", "size": "0.25rem" },
				{ "slug": "20", "name": "2", "size": "0.5rem" },
				{ "slug": "30", "name": "3", "size": "1rem" },
				{ "slug": "40", "name": "4", "size": "1.5rem" },
				{ "slug": "50", "name": "5", "size": "2.5rem" },
				{ "slug": "60", "name": "6", "size": "4rem" },
				{ "slug": "70", "name": "7", "size": "6rem" },
				{ "slug": "80", "name": "8", "size": "8rem" }
			]
		}
	},
	"styles": {
		"spacing": { "blockGap": "var(--wp--preset--spacing--40)" }
	}
}
```

If the brand palette uses its own names, add `base`/`contrast`/`contrast-2` as extra entries with the same hex values as the brand colors they stand for, so the editor's color picker shows real swatches.

## 2. WooCommerce's own variables (`--wc-*`)

### Global, on `:root` (from `assets/css/woocommerce.css`)

`woocommerce.css` (handle `woocommerce-general`) also loads on block themes unless you remove it with the `woocommerce_enqueue_styles` filter. It defines these on `:root`, with WooCommerce's purple identity as defaults:

| Variable | Default | What it colors |
|---|---|---|
| `--woocommerce`, `--wc-primary` | `#720eec` | Primary accent (buttons/links in WC-rendered markup) |
| `--wc-primary-text` | `#fcfbfe` | Text on primary |
| `--wc-secondary` / `--wc-secondary-text` | `#e9e6ed` / `#515151` | Secondary surfaces |
| `--wc-highlight` / `--wc-highligh-text` (sic) | `#958e09` / `white` | Highlights (e.g. sale) |
| `--wc-content-bg` | `#fff` | Content background |
| `--wc-subtext` | `#767676` | Muted text |
| `--wc-green` / `--wc-red` / `--wc-orange` / `--wc-blue` | `#007518` / `#a00` / `#ffba00` / `#2ea2cc` | Status colors: in stock, errors/required, etc. (`--wc-red` is the most used) |
| `--wc-destructive` | `#cc1818` | Destructive actions |
| `--wc-form-border-color` | `rgba(32, 7, 7, 0.8)` | Form fields |
| `--wc-form-border-radius` / `--wc-form-border-width` | `4px` / `1px` | Form fields |
| `--wc-form-color-text` / `--wc-form-color-background` | (fallbacks `#444`/`#000` and `#fff`) | Form fields; consumed but not defined on `:root` |
| `--wc-card-border-radius` | `8px` | Cards |

From the block styles, also on `:root`:

| Variable | Default | What it colors |
|---|---|---|
| `--wc-separator-color` | `color-mix(in srgb, currentColor 20%, transparent)` | Borders and separators in cart, checkout, Mini-Cart and other blocks (the most used WC variable) |

Override them once in the theme, mapped to the preset variables `theme.json` already renders, so the brand lives in one place. Two rules:

- **Map with `var()`**: `--wc-primary: var(--wp--preset--color--contrast);`. A bare `--wc-primary: --wp--preset--color--contrast` is an invalid value.
- **Declare them on `body`, not `:root`.** WordPress's global styles and `woocommerce.css` are both enqueued on `wp_enqueue_scripts` at the default priority, so print order isn't guaranteed. With equal `:root` selectors, WooCommerce's purple defaults can win. A declaration on `body` wins for everything inside it regardless of order, because each element inherits from its nearest ancestor.

No extra stylesheet is needed: put the mapping in `theme.json` under `styles.css` (WP 6.2+). WordPress appends it to the global styles stylesheet, next to the `--wp--preset--*` variables. `settings.custom` can't do it, because it only emits `--wp--custom--*` names.

```json
{
	"version": 3,
	"styles": {
		"css": "body{--wc-primary:var(--wp--preset--color--contrast);--woocommerce:var(--wc-primary);--wc-primary-text:var(--wp--preset--color--base);--wc-content-bg:var(--wp--preset--color--base);--wc-subtext:var(--wp--preset--color--contrast-2);--wc-form-border-color:var(--wp--preset--color--contrast-2);--wc-separator-color:color-mix(in srgb,currentColor 15%,transparent)}"
	}
}
```

The same mapping written as readable CSS. Use this form in a theme stylesheet instead, if you prefer a file:

```css
body {
	--wc-primary: var(--wp--preset--color--contrast);
	--woocommerce: var(--wc-primary);
	--wc-primary-text: var(--wp--preset--color--base);
	--wc-secondary: var(--wp--preset--color--base);
	--wc-secondary-text: var(--wp--preset--color--contrast);
	--wc-highlight: var(--wp--preset--color--contrast);
	--wc-highligh-text: var(--wp--preset--color--base);
	--wc-content-bg: var(--wp--preset--color--base);
	--wc-subtext: var(--wp--preset--color--contrast-2);
	--wc-red: #b3261e;                 /* errors / required markers: keep it readable on base */
	--wc-green: #1b6e3a;
	--wc-form-border-color: var(--wp--preset--color--contrast-2);
	--wc-form-border-radius: 4px;
	--wc-form-color-text: var(--wp--preset--color--contrast);
	--wc-form-color-background: var(--wp--preset--color--base);
	--wc-card-border-radius: 8px;
	--wc-separator-color: color-mix(in srgb, currentColor 15%, transparent);
}
```

Check contrast for `--wc-red`/`--wc-green` against `base` (WCAG AA 4.5:1 for text).

### Block-scoped (set by the block, override per block only)

These are set inline from block attributes, or on the block's own wrapper. Don't set them globally on `:root`. Style the block instead, or let editors choose in the block's settings.

| Variable | Set by | Default / source |
|---|---|---|
| `--wc-product-filters-background-color`, `--wc-product-filters-text-color`, `--wc-product-filter-block-spacing` | Product Filters block (PHP render, from its color and block-gap attributes) | attribute values |
| `--wc-product-filters-overlay-background-color`, `-overlay-text-color`, `-overlay-icon-size` | Never set by WooCommerce: theme hooks for the mobile filters drawer | fall back to the filter's colors, then `Canvas` / `CanvasText`; icon `1.5em` |
| `--wc-product-filters-closed-translate-x` | Product Filters overlay CSS | `-100%` / `100%` (left/right drawer) |
| `--wc-product-filter-chips-*`, `--wc-product-filter-removable-chips-*`, `--wc-product-filter-checkbox-list-*`, `--wc-product-filter-price-slider*` | Filter Chips, Removable Chips, Checkbox List, Price Slider (from their color attributes) | attribute values |
| `--wc-block-product-gallery-large-image-ratio-width` / `-height` | Product Gallery (aspect ratio attribute) | `1` / `1` |
| `--wc-block-product-gallery-thumbnails-size` | Product Gallery Thumbnails (size attribute) | attribute value |
| `--wc-customer-account-*` (`viewport-gutter`, `dropdown-surface`, `dropdown-padding-inline/-block`, `dropdown-border-color`, `dropdown-item-hover-bg`) | Customer Account block wrapper | built from `spacing--20/30`, `color--base`, `currentColor` mixes |
| `--drawer-width`, `--neg-drawer-width` | Mini-Cart Contents (PHP prints them on `:root` from the block's width setting) | block attribute |

Class modifiers such as `wc-block-product-filter--hidden` or `wc-block-customer-account--is-dropdown-open` look similar but are BEM class names, not variables.

## 3. Templates and patterns: what they assume

- **Block templates and parts** reference `small`, `medium`, `large`, spacing `20` and color `contrast`. A theme that overrides a WooCommerce template (e.g. its own `templates/single-product.html`) replaces this, but WooCommerce's fallback templates and parts (`mini-cart.html`, `checkout-header.html`, the Add to Cart + Options parts) still use those slugs.
- **WooCommerce patterns** (footers, coming-soon, product sections) lean on spacing `10`–`50`, `small`/`medium`, `contrast`/`contrast-2`, and also hard-code some pixel font sizes. Expect to restyle patterns you insert; the preset slugs above make most of them adapt.
- **Emails** don't use these variables. They're styled by WooCommerce's email settings and the email editor; see `emails.md`.

## 4. Re-auditing after a WooCommerce update

Run from the WooCommerce plugin directory. The output lists every preset and `--wc-*` variable consumed, with counts.

```bash
# Preset variables consumed by block styles (frontend), with fallbacks:
cat assets/client/blocks/*.css | grep -v rtl | grep -oE "var\(--wp--preset--[a-z-]+--[a-z0-9-]+(,[^)]{1,40})?" | sort | uniq -c | sort -rn

# Preset slugs referenced by block templates, parts and patterns:
grep -rhoE 'var:preset\|[a-z-]+\|[a-z0-9-]+|var\(--wp--preset--[a-z-]+--[a-z0-9-]+|"(textColor|backgroundColor|fontSize|fontFamily)":"[a-z0-9-]+"' templates/templates templates/parts patterns | sort | uniq -c | sort -rn

# WooCommerce's own variables: consumed, and defined (with defaults):
cat assets/client/blocks/*.css assets/css/woocommerce*.css | grep -oE "var\(--wc-[a-z0-9-]+" | sort | uniq -c | sort -rn
grep -ohE ":root\{[^}]*\}" assets/css/woocommerce.css assets/client/blocks/*.css | sort -u
```

## Common mistakes

- A palette with only brand slugs (`primary`, `ink`…) and no `base`/`contrast`/`contrast-2`. The SKU prefix, order-summary badges and pattern text lose their color, or fall back to white/black.
- Custom spacing slugs (`sm`, `md`, `lg`) instead of `10`–`80`. Product Filters overlay padding (`spacing--30`, no fallback) collapses, and patterns lose their spacing.
- `defaultFontSizes: false` without defining `small`, `medium`, `large`, `x-large`. Cart/checkout text falls to `14px`/`22px` fallbacks, and the coming-soon title has no size at all. Same with `defaultSpacingSizes: false`.
- No `x-small`. WooCommerce computes `small × 0.875`, which may not match the type scale.
- Leaving WooCommerce's purple defaults (`--wc-primary: #720eec`, `--wc-highlight: #958e09`) because the theme never overrides `--wc-*`.
- Overriding `--wc-*` on `:root` and seeing no effect: WooCommerce's own `:root` block printed later wins. Declare them on `body`.
- Setting block-scoped variables (`--wc-product-filter-chips-*`, gallery ratios) on `:root`. They are per-block attribute values; a global value fights the editor settings.
- Hard-coding colors in theme CSS targeting `.wc-block-components-*` instead of overriding the variables and presets WooCommerce already reads.
