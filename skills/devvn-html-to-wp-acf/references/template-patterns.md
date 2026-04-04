# Template Conversion Patterns

> All patterns below use `get_field()`, `have_rows()`, `get_sub_field()` — these functions are provided by both ACF and SCF (Secure Custom Fields). The `function_exists()` guards ensure graceful fallback if neither plugin is active.

## Template header

Every converted page template starts with:

```php
<?php
/*
Template Name: {Display Name}
*/

// Helper: get ACF field with fallback.
function {prefix}_field( $name, $default = '' ) {
    if ( ! function_exists( 'get_field' ) ) {
        return $default;
    }
    $val = get_field( $name );
    return ( $val !== null && $val !== '' && $val !== false ) ? $val : $default;
}

// Shared variables used across sections.
$logo_url   = {prefix}_field( 'logo', get_template_directory_uri() . '/default-logo.png' );
$phone      = {prefix}_field( 'contact_phone', '0123 456 789' );
$phone_raw  = preg_replace( '/[^0-9]/', '', $phone );
$email      = {prefix}_field( 'contact_email', 'info@example.com' );
$address    = {prefix}_field( 'contact_address', 'Address here' );
?>
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- No <title> or <meta description> — SEO plugin handles these -->
    ...
    <?php wp_head(); ?>
</head>
```

## Conversion pattern: Simple text

**Before (HTML):**
```html
<h2>Hardcoded Title</h2>
<p>Hardcoded description text.</p>
```

**After (PHP):**
```php
<h2><?php echo esc_html( {prefix}_field( 'section_title', 'Hardcoded Title' ) ); ?></h2>
<p><?php echo esc_html( {prefix}_field( 'section_desc', 'Hardcoded description text.' ) ); ?></p>
```

## Conversion pattern: Image

**Before:**
```html
<img src="photo.jpg" alt="Description">
```

**After:**
```php
<?php $img = {prefix}_field( 'image_field', 'photo.jpg' ); ?>
<img src="<?php echo esc_url( $img ); ?>" alt="Description">
```

## Conversion pattern: Background image

**Before (in CSS):**
```css
.hero { background: url('hero.jpg') center/cover; }
```

**After (move to inline style):**
```php
<?php $hero_bg = {prefix}_field( 'hero_bg', 'hero.jpg' ); ?>
<section class="hero" style="background: linear-gradient(...), url('<?php echo esc_url( $hero_bg ); ?>') center/cover no-repeat;">
```

And simplify the CSS rule:
```css
.hero { background-size: cover; background-position: center; background-repeat: no-repeat; }
```

## Conversion pattern: Repeater with fallback

**Before:**
```html
<div class="grid">
    <div class="card"><h3>Card 1</h3><p>Desc 1</p></div>
    <div class="card"><h3>Card 2</h3><p>Desc 2</p></div>
    <div class="card"><h3>Card 3</h3><p>Desc 3</p></div>
</div>
```

**After:**
```php
<div class="grid">
    <?php if ( function_exists( 'have_rows' ) && have_rows( 'cards' ) ) : ?>
        <?php while ( have_rows( 'cards' ) ) : the_row(); ?>
            <div class="card">
                <h3><?php echo esc_html( get_sub_field( 'title' ) ); ?></h3>
                <p><?php echo esc_html( get_sub_field( 'desc' ) ); ?></p>
            </div>
        <?php endwhile; ?>
    <?php else : ?>
        <!-- Original HTML as fallback -->
        <div class="card"><h3>Card 1</h3><p>Desc 1</p></div>
        <div class="card"><h3>Card 2</h3><p>Desc 2</p></div>
        <div class="card"><h3>Card 3</h3><p>Desc 3</p></div>
    <?php endif; ?>
</div>
```

## Conversion pattern: Repeater with counter

For numbered items, alternating layouts (zigzag), or dividers between items:

```php
<?php if ( function_exists( 'have_rows' ) && have_rows( 'items' ) ) : $i = 0; ?>
    <?php while ( have_rows( 'items' ) ) : the_row(); $i++; ?>
        <?php if ( $i > 1 ) : ?><div class="divider"></div><?php endif; ?>
        <div class="item <?php echo ( $i % 2 === 0 ) ? 'reverse' : ''; ?>">
            <span class="number"><?php echo str_pad( $i, 2, '0', STR_PAD_LEFT ); ?></span>
            ...
        </div>
    <?php endwhile; ?>
<?php endif; ?>
```

## Conversion pattern: Textarea as list

When a repeater is overkill for a simple bullet list, use textarea with one item per line:

**ACF field:** `textarea` with instructions "Mỗi dòng = 1 mục"

**Template:**
```php
<?php $items = get_sub_field( 'features' ); if ( $items ) : ?>
    <ul>
        <?php foreach ( explode( "\n", trim( $items ) ) as $item ) : if ( trim( $item ) ) : ?>
            <li><?php echo esc_html( trim( $item ) ); ?></li>
        <?php endif; endforeach; ?>
    </ul>
<?php endif; ?>
```

## Conversion pattern: WYSIWYG for link groups

For footer menu columns or legal links where the user needs to edit both text and URLs:

```php
<?php echo wp_kses_post( {prefix}_field( 'footer_legal', '<a href="#">Policy</a> <a href="#">Terms</a>' ) ); ?>
```

## Conversion pattern: Phone number

Store formatted number, derive raw digits:

```php
// At template top:
$phone     = {prefix}_field( 'phone', '0972 220 777' );
$phone_raw = preg_replace( '/[^0-9]/', '', $phone );

// In template:
<a href="tel:<?php echo esc_attr( $phone_raw ); ?>"><?php echo esc_html( $phone ); ?></a>
```

## Conversion pattern: Forms (shortcode)

HTML forms (`<form>`) should **not** be hardcoded in the template. Instead, use a `textarea` ACF field where the user pastes a shortcode from their form plugin (Contact Form 7, WPForms, Fluent Forms, Gravity Forms, etc.).

**ACF field:**
```php
array(
    'key'          => 'field_{prefix}_form_shortcode',
    'label'        => 'Form shortcode',
    'name'         => '{prefix}_form_shortcode',
    'type'         => 'textarea',
    'rows'         => 3,
    'instructions' => 'Paste form plugin shortcode. E.g.: [contact-form-7 id="123"]',
),
```

**Template:**
```php
<?php echo do_shortcode( {prefix}_field( 'form_shortcode', '' ) ); ?>
```

Key rules:
- Remove all hardcoded `<form>` HTML from the template
- Remove any associated demo/mock JS for form submission
- Use `do_shortcode()` to render — this lets any form plugin work
- The field is `textarea` (not `text`) so multiline shortcodes or multiple shortcodes are supported
- If the page has multiple forms (e.g., hero form + contact form), create separate shortcode fields for each

## Escaping rules

| Context | Function | Example |
|---------|----------|---------|
| Inside HTML text | `esc_html()` | `<h2><?php echo esc_html( $val ); ?></h2>` |
| Inside `href`/`src` | `esc_url()` | `<a href="<?php echo esc_url( $url ); ?>">` |
| Inside HTML attributes | `esc_attr()` | `<div data-x="<?php echo esc_attr( $val ); ?>">` |
| Rich HTML (trusted) | `wp_kses_post()` | `<?php echo wp_kses_post( $wysiwyg ); ?>` |
| Never | raw `echo get_field()` | Security vulnerability |
