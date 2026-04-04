# Asset Stripping for Page Speed

## Purpose

Landing pages should load only their own inline CSS and JS — no WordPress core styles, theme CSS, plugin scripts, emoji detection, or block editor global styles. However, SEO plugin output (meta tags, canonical, schema) and form plugin scripts (CF7, WPForms, etc.) must be preserved.

## functions.php: Strip assets for the template

```php
add_action( 'template_redirect', '{prefix}_strip_assets_on_landing' );

function {prefix}_strip_assets_on_landing() : void {
    if ( ! is_page_template( '{template-file}.php' ) ) {
        return;
    }

    // Disable admin bar on landing page.
    add_filter( 'show_admin_bar', '__return_false' );

    // 1. Dequeue all scripts & styles, keep form plugin + full dependency tree.
    add_action( 'wp_enqueue_scripts', function () {
        global $wp_scripts, $wp_styles;

        // Scripts to keep (add form plugin handles here).
        $keep_roots = array( 'contact-form-7', 'wpcf7-recaptcha', 'swv' );
        if ( $wp_scripts instanceof WP_Scripts ) {
            // Recursively resolve all dependencies.
            $keep_all = array();
            $resolve  = function ( $handle ) use ( $wp_scripts, &$keep_all, &$resolve ) {
                if ( in_array( $handle, $keep_all, true ) ) {
                    return;
                }
                $keep_all[] = $handle;
                if ( isset( $wp_scripts->registered[ $handle ] ) ) {
                    foreach ( $wp_scripts->registered[ $handle ]->deps as $dep ) {
                        $resolve( $dep );
                    }
                }
            };
            foreach ( $keep_roots as $h ) {
                $resolve( $h );
            }
            foreach ( $wp_scripts->queue as $handle ) {
                if ( in_array( $handle, $keep_all, true ) ) {
                    continue;
                }
                wp_dequeue_script( $handle );
                wp_deregister_script( $handle );
            }
        }

        // Remove all styles (landing page uses its own inline CSS).
        if ( $wp_styles instanceof WP_Styles ) {
            foreach ( $wp_styles->queue as $handle ) {
                wp_dequeue_style( $handle );
                wp_deregister_style( $handle );
            }
        }
    }, 9999 );

    // 2. Block wp_head from printing scripts & styles (keep footer for form plugin JS).
    remove_action( 'wp_head', 'wp_print_styles', 8 );
    remove_action( 'wp_head', 'wp_print_head_scripts', 9 );

    // 3. Remove emoji JS & CSS.
    remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
    remove_action( 'wp_print_styles', 'print_emoji_styles' );

    // 4. Remove block editor global styles & SVG filters.
    remove_action( 'wp_enqueue_scripts', 'wp_enqueue_global_styles' );
    remove_action( 'wp_body_open', 'wp_global_styles_render_svg_filters' );
    remove_action( 'wp_footer', 'wp_enqueue_global_styles', 1 );

    // 5. Remove oEmbed JS (keep discovery link for SEO).
    remove_action( 'wp_head', 'wp_oembed_add_host_js' );

    // 6. Remove Flatsome theme inline CSS (custom-css).
    remove_action( 'wp_head', 'flatsome_custom_css', 100 );
    remove_action('wp_footer', 'flatsome_mobile_menu', 7);
}
```

### Key points about dependency resolution

Form plugins like CF7 depend on WordPress core scripts (`wp-i18n` → `wp-hooks`, etc.). If you only keep the top-level handle and strip the rest, you get runtime errors like:

> `Uncaught TypeError: Cannot read properties of undefined (reading 'i18n')`

The recursive `$resolve` function walks the full dependency tree so nothing is missed. To add another form plugin (e.g., WPForms), simply add its handles to `$keep_roots`.

### Do NOT remove `wp_print_footer_scripts`

Earlier versions of this pattern included:
```php
remove_action( 'wp_footer', 'wp_print_footer_scripts', 20 ); // DON'T do this
```
This prevents form plugin scripts from being printed in the footer. Remove only `wp_head` print actions; the footer filter in the template handles the rest.

## What wp_head() still outputs (and why we want it)

After stripping, `wp_head()` will only output:
- SEO plugin meta tags (Yoast, RankMath, SEOPress)
- Canonical URL
- Open Graph / Twitter Card meta
- JSON-LD schema
- RSS feed discovery links
- oEmbed discovery links
- `<link rel="shortlink">`

These are all beneficial for SEO and should be kept.

## wp_footer() filtering in the template

Even after stripping in `functions.php`, some plugins may inject unwanted CSS/JS via `wp_footer()`. Filter the output while keeping form plugin scripts:

```php
<?php
ob_start();
wp_footer();
$wp_footer_output = ob_get_clean();
// Remove leftover stylesheets.
$wp_footer_output = preg_replace( '/<link[^>]*rel=[\'"]stylesheet[\'"][^>]*>/i', '', $wp_footer_output );
// Remove scripts, keep performance plugins + form plugin + WP core deps.
$keep_js = array( 'rocket', 'lazyload', 'contact-form-7', 'wpcf7', 'swv', 'wp-i18n', 'wp-hooks', 'wp-polyfill', 'wp-includes' );
$wp_footer_output = preg_replace_callback(
    '/<script\b[^>]*>.*?<\/script>/si',
    function( $m ) use ( $keep_js ) {
        foreach ( $keep_js as $k ) {
            if ( stripos( $m[0], $k ) !== false ) {
                return $m[0];
            }
        }
        return '';
    },
    $wp_footer_output
);
echo $wp_footer_output;
?>
```

The `$keep_js` array matches against the script tag content (src URL or inline code). `'wp-includes'` catches all WP core dependency scripts loaded from `/wp-includes/js/dist/`.

### Adapting for other form plugins

| Plugin | Handles to add to `$keep_roots` | Keywords for `$keep_js` |
|--------|------|---------|
| Contact Form 7 | `contact-form-7`, `wpcf7-recaptcha`, `swv` | `contact-form-7`, `wpcf7`, `swv` |
| WPForms | `wpforms` | `wpforms` |
| Fluent Forms | `fluentform-advanced` | `fluentform` |
| Gravity Forms | `gform_gravityforms` | `gravityforms` |

## Form plugin CSS override

When stripping all styles, form plugins lose their default styling. Add CSS overrides in the template `<style>` block to match the landing page design. Example for CF7:

```css
/* CF7 base inputs */
.wpcf7-form input[type="text"],
.wpcf7-form input[type="email"],
.wpcf7-form input[type="tel"],
.wpcf7-form select,
.wpcf7-form textarea {
    width: 100%; padding: 0.75rem 1rem; border-radius: var(--radius-xl);
    font-size: 0.875rem; background: rgba(255,255,255,0.95);
    border: 1px solid rgba(59,45,139,0.15); transition: all 0.3s ease;
    font-family: inherit; box-sizing: border-box;
}
.wpcf7-form input:focus,
.wpcf7-form textarea:focus { outline: none; border-color: var(--brand); }

/* CF7 submit button — style to match landing page CTA */
.wpcf7-form .wpcf7-submit {
    width: 100%; padding: 0.875rem 2rem; border-radius: var(--radius-full);
    background: linear-gradient(135deg, var(--brand), var(--brand-light));
    color: #fff; font-weight: 600; cursor: pointer; border: none;
}

/* CF7 validation & response */
.wpcf7-not-valid { border-color: #dc2626; }
.wpcf7-not-valid-tip { font-size: 0.75rem; color: #dc2626; }
.wpcf7-form.sent .wpcf7-response-output { background: #f0fdf4; color: #15803d; }
.wpcf7-form.invalid .wpcf7-response-output { background: #fef2f2; color: #dc2626; }

/* Dark background form (e.g., hero section) */
.hero-form-glass .wpcf7-form label { color: rgba(255,255,255,0.85); }
.hero-form-glass .wpcf7-form .wpcf7-submit {
    background: var(--dark-950); border-radius: var(--radius-xl);
}
```

## Classic Editor for ACF templates

Block themes use Gutenberg by default, which doesn't work well with ACF field groups. Force Classic Editor for specific templates:

```php
add_filter( 'use_block_editor_for_post', function ( $use, $post ) {
    if ( get_page_template_slug( $post ) === '{template-file}.php' ) {
        return false;
    }
    return $use;
}, 10, 2 );
```

## CSS minification

Inline CSS in the template should be minified at runtime:

```php
<style><?php ob_start(); ?>
    /* Full readable CSS here */
<?php
$css = ob_get_clean();
// Remove comments.
$css = preg_replace('!/\*[^*]*\*+([^/][^*]*\*+)*/!', '', $css);
// Collapse whitespace.
$css = preg_replace('/\s+/', ' ', $css);
// Remove spaces around delimiters.
$css = preg_replace('/\s*([{}:;,])\s*/', '$1', $css);
// Remove trailing semicolons before closing braces.
$css = str_replace(';}', '}', $css);
echo trim($css);
?></style>
```

## Multiple landing pages

If you have multiple landing page templates, extend `is_page_template()` with an array:

```php
if ( ! is_page_template( array( 'ldp.php', 'campaign.php', 'promo.php' ) ) ) {
    return;
}
```
