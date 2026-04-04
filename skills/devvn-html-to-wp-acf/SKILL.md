---
name: devvn-html-to-wp-acf
description: "Convert static HTML landing pages into WordPress page templates with ACF (Advanced Custom Fields) or SCF (Secure Custom Fields) integration. Covers field registration via PHP, template conversion with get_field/have_rows, CSS minification, asset stripping, and page speed optimization."
compatibility: "WordPress 6.0+ with ACF Pro or SCF (Secure Custom Fields). PHP 7.4+. Works with any theme (classic or block)."
---

# HTML to WordPress + ACF/SCF Conversion

> **ACF vs SCF**: Secure Custom Fields (SCF) is the WordPress.org-maintained fork of ACF. Both share the same API — `get_field()`, `have_rows()`, `acf_add_local_field_group()`, etc. All code in this skill works with either plugin without modification. When this document says "ACF", it applies equally to SCF.

## When to use

Use this skill when converting a static HTML page (landing page, one-page site, campaign page) into a WordPress page template that uses ACF or SCF for dynamic content management:

- Converting a standalone `.html` file into a `.php` page template
- Making hardcoded text, images, repeaters editable via ACF admin
- Registering ACF field groups in PHP (no JSON import needed)
- Optimizing the converted page for speed (inline CSS minification, stripping unnecessary WP assets)
- Setting up Classic Editor for specific page templates in a block theme

## Inputs required

- The source HTML file (or existing `.php` template with hardcoded content)
- The target WordPress theme directory
- Whether ACF Pro is available (repeater, flexible content require Pro)
- Any existing `functions.php` to extend

## Procedure

### 1) Analyze the HTML structure

1. Read the full HTML file and identify every content section
2. Map each section to a category:
   - **Simple fields**: headings, descriptions, badge text, phone, email, address, URLs
   - **Repeaters**: lists of cards, features, FAQ items, stats, gallery items, menu columns
   - **Images**: hero backgrounds, product photos, icons, gallery
   - **Rich text**: content blocks with links (footer menus, legal text)
3. Group fields into logical **tabs** matching the page sections (Hero, Features, Products, Services, FAQ, Contact, Footer, etc.)
4. Note which sections need **fallback HTML** (default content when ACF fields are empty)

Read: references/field-mapping.md

### 2) Create the ACF field registration file

1. Create `acf-{template-name}-fields.php` in the theme root
2. Use `acf_add_local_field_group()` inside an `acf/init` action hook
3. Structure fields with tabs for each section
4. Apply these rules:
   - Guard with `if ( ! function_exists( 'acf_add_local_field_group' ) ) { return; }`
   - Use descriptive, prefixed keys: `field_{prefix}_{section}_{name}`
   - Use descriptive, prefixed names: `{prefix}_{field_name}`
   - Set `default_value` for all text/textarea fields with the original hardcoded content
   - Use `return_format => 'url'` for images (simpler output)
   - Use `repeater` for any list of items (cards, features, FAQ, stats)
   - Use `wysiwyg` with `toolbar => 'basic'` and `media_upload => 0` for rich link content (footer menus, legal text)
   - Set location rule to match the page template file
5. Include the file from `functions.php` with `require_once get_theme_file_path()`

Read: references/acf-registration.md

### 3) Create helper functions in the template

Add these at the top of the page template file, before any HTML output:

```php
// Helper: get ACF field with fallback.
function {prefix}_field( $name, $default = '' ) {
    if ( ! function_exists( 'get_field' ) ) {
        return $default;
    }
    $val = get_field( $name );
    return ( $val !== null && $val !== '' && $val !== false ) ? $val : $default;
}
```

Also declare shared variables used across multiple sections (phone, email, address, logo URL) at the top so both contact and footer sections can reference them.

### 4) Convert HTML sections to ACF-powered PHP

For each section in the HTML, apply these conversion patterns:

**Simple text fields:**
```php
<?php echo esc_html( {prefix}_field( 'field_name', 'Default text' ) ); ?>
```

**Image fields:**
```php
<?php $img = {prefix}_field( 'field_name', 'fallback-url' ); ?>
<img src="<?php echo esc_url( $img ); ?>" alt="...">
```

**Repeater fields:**
```php
<?php if ( function_exists( 'have_rows' ) && have_rows( 'repeater_name' ) ) : ?>
    <?php while ( have_rows( 'repeater_name' ) ) : the_row(); ?>
        <div><?php echo esc_html( get_sub_field( 'sub_field' ) ); ?></div>
    <?php endwhile; ?>
<?php else : ?>
    <!-- Original hardcoded HTML as fallback -->
<?php endif; ?>
```

**WYSIWYG fields (rich text with links):**
```php
<?php echo wp_kses_post( {prefix}_field( 'field_name', '<a href="#">Default</a>' ) ); ?>
```

**Inline background images:**
```php
style="background: linear-gradient(...), url('<?php echo esc_url( $bg_url ); ?>') center/cover no-repeat;"
```

**Form shortcodes (replace hardcoded `<form>`):**
```php
<?php echo do_shortcode( {prefix}_field( 'form_shortcode', '' ) ); ?>
```
Use a `textarea` ACF field so users paste shortcodes from their form plugin (CF7, WPForms, Fluent Forms, etc.). Remove all hardcoded `<form>` HTML and any associated demo JS.

Key rules:
- Always provide fallback content (the original HTML) so the page works without ACF data
- Always escape output: `esc_html()` for text, `esc_url()` for URLs, `esc_attr()` for attributes, `wp_kses_post()` for rich HTML
- Check `function_exists('have_rows')` before any repeater loop
- For repeaters with counters/alternating layout, use a `$i` counter variable

Read: references/template-patterns.md

### 5) Optimize CSS delivery

1. Move hardcoded CSS background URLs out of `<style>` into inline `style=""` attributes on elements (so they can be dynamic)
2. Wrap the `<style>` block with output buffering and minify:

```php
<style><?php ob_start(); ?>
    /* ... all CSS ... */
<?php
$css = ob_get_clean();
$css = preg_replace('!/\*[^*]*\*+([^/][^*]*\*+)*/!', '', $css);
$css = preg_replace('/\s+/', ' ', $css);
$css = preg_replace('/\s*([{}:;,])\s*/', '$1', $css);
$css = str_replace(';}', '}', $css);
echo trim($css);
?></style>
```

### 6) Integrate with WordPress head/footer and strip unnecessary assets

1. Add `<?php wp_head(); ?>` before `</head>` — this outputs SEO plugin meta tags (Yoast, RankMath, etc.)
2. Remove any hardcoded `<title>` and `<meta name="description">` — let SEO plugins handle these
3. Add filtered `wp_footer()` before `</body>` — keep form plugin scripts + performance plugins:

```php
<?php
ob_start();
wp_footer();
$wp_footer_output = ob_get_clean();
$wp_footer_output = preg_replace( '/<link[^>]*rel=[\'"]stylesheet[\'"][^>]*>/i', '', $wp_footer_output );
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

4. In `functions.php`, add an asset stripping function on `template_redirect` that **recursively resolves** form plugin dependencies (so `wp-i18n`, `wp-hooks`, etc. are kept automatically):
5. **Important**: Do NOT `remove_action( 'wp_footer', 'wp_print_footer_scripts' )` — this blocks form plugin JS from loading
6. Add CSS overrides for the form plugin in the template `<style>` block to match the landing page design (inputs, submit button, validation messages, dark-background variants)

Read: references/asset-stripping.md

### 7) Additional WordPress integrations

1. **Classic Editor for the template** — If the theme uses block editor but ACF works better with Classic Editor:
```php
add_filter( 'use_block_editor_for_post', function ( $use, $post ) {
    if ( get_page_template_slug( $post ) === '{template-file}.php' ) {
        return false;
    }
    return $use;
}, 10, 2 );
```

2. **Disable admin bar** on the landing page:
```php
add_filter( 'show_admin_bar', '__return_false' );
```

3. **Remove old ACF JSON export** if one was created during development — the PHP registration replaces it.

## Verification

- Page renders identically to the original HTML when no ACF data is entered (fallback mode)
- All sections update correctly when ACF fields are filled in the admin
- View source shows no unnecessary CSS/JS from WordPress core, theme, or plugins
- SEO plugin meta tags (canonical, og:tags, schema) appear in `<head>`
- No admin bar visible on the frontend
- Page loads with Classic Editor in wp-admin (if configured)
- Repeater fields add/remove items correctly and the page reflects changes
- All output is properly escaped (no raw `echo get_field()` without escaping)

## Failure modes / debugging

| Symptom | Cause | Fix |
|---------|-------|-----|
| ACF fields don't appear in admin | Location rule doesn't match template | Check `post_template` value matches the `.php` filename exactly |
| Fields appear but page shows defaults | `get_field()` returns `false`/empty | Ensure field names in PHP match ACF registration `name` values |
| Repeater shows nothing | `have_rows()` returns false | Check repeater field `name` matches, verify sub_field names |
| Page shows WP theme CSS/JS | Asset stripping not running | Verify `is_page_template()` check matches template filename |
| Broken layout after conversion | CSS background URLs still hardcoded in `<style>` | Move dynamic backgrounds to inline `style=""` on elements |
| `function_exists('get_field')` returns false | ACF/SCF not active | The fallback pattern handles this — page uses default HTML. Works with both ACF and SCF |
| Repeater fields missing in SCF Free | Using ACF Free (not Pro) | SCF includes repeater for free; ACF Free does not. Switch to SCF or upgrade to ACF Pro |
| Images not showing | Return format mismatch | Use `return_format => 'url'` for image fields used in `src=""` |
| Footer variables undefined | Variables declared in wrong scope | Declare shared variables (`$phone`, `$email`, etc.) at the top of the template |
| Form plugin JS error: `Cannot read 'i18n'` | Core dependency scripts stripped | Use recursive dependency resolution in asset stripping; do NOT remove `wp_print_footer_scripts` |
| Form not submitting / no AJAX | Form plugin script not loaded | Add plugin handles to `$keep_roots` in functions.php and keywords to `$keep_js` in footer filter |
| Form unstyled after conversion | Plugin CSS stripped with all other styles | Add CSS overrides for form plugin classes (`.wpcf7-form`, etc.) in the template `<style>` block |
| Form shortcode shows raw text | `do_shortcode()` not wrapping the field output | Ensure template uses `echo do_shortcode( {prefix}_field( 'form_shortcode', '' ) )` |
| intelephense "undefined function" errors | IDE missing WordPress stubs | These are IDE-only — runtime works fine. Install `php-stubs/wordpress-stubs` if needed |

## Escalation

- If the HTML uses JavaScript frameworks (React, Vue) for rendering — this skill covers static HTML only
- If multi-language support is required — coordinate with WPML/Polylang setup
- If the page template needs to work across multiple themes — consider making it a plugin instead
