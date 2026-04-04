# ACF/SCF Field Registration via PHP

> **ACF and SCF compatibility**: Secure Custom Fields (SCF) is the WordPress.org fork of ACF. Both use identical PHP APIs: `acf_add_local_field_group()`, `get_field()`, `have_rows()`, `get_sub_field()`, etc. The function guard `function_exists( 'acf_add_local_field_group' )` works for both plugins. All code below is compatible with ACF Pro, ACF Free, and SCF without changes.

## Why PHP over JSON import

- **Auto-registers** when the theme is active — no manual import step
- **Version controlled** alongside the theme code
- **Portable** — works on any WordPress install with ACF or SCF active
- Fields can still be exported as JSON from ACF/SCF > Tools > Export if needed later

## File structure

Create `acf-{template-name}-fields.php` in the theme root:

```php
<?php
/**
 * ACF Fields for {Template Name}
 */

if ( ! function_exists( 'acf_add_local_field_group' ) ) {
    return;
}

add_action( 'acf/init', function () {
    acf_add_local_field_group( array(
        'key'      => 'group_{unique_id}',
        'title'    => '{Template Display Name}',
        'fields'   => array(
            // Tabs and fields here...
        ),
        'location' => array(
            array(
                array(
                    'param'    => 'post_template',
                    'operator' => '==',
                    'value'    => '{template-file}.php',
                ),
            ),
        ),
        'menu_order'            => 0,
        'position'              => 'normal',
        'style'                 => 'default',
        'label_placement'       => 'top',
        'instruction_placement' => 'label',
        'active'                => true,
    ) );
} );
```

## Include from functions.php

```php
require_once get_theme_file_path( 'acf-{template-name}-fields.php' );
```

## Tab field pattern

```php
array(
    'key'       => 'field_{prefix}_tab_{section}',
    'label'     => 'Section Name',
    'type'      => 'tab',
    'placement' => 'left',
),
```

## Common field patterns

**Text with default:**
```php
array(
    'key'           => 'field_{prefix}_{name}',
    'label'         => 'Label',
    'name'          => '{prefix}_{name}',
    'type'          => 'text',
    'default_value' => 'Original hardcoded text',
),
```

**Textarea:**
```php
array(
    'key'           => 'field_{prefix}_{name}',
    'label'         => 'Label',
    'name'          => '{prefix}_{name}',
    'type'          => 'textarea',
    'rows'          => 3,
    'default_value' => 'Original paragraph text...',
),
```

**Image:**
```php
array(
    'key'           => 'field_{prefix}_{name}',
    'label'         => 'Label',
    'name'          => '{prefix}_{name}',
    'type'          => 'image',
    'return_format' => 'url',
    'preview_size'  => 'medium',
),
```

**Select:**
```php
array(
    'key'           => 'field_{prefix}_{name}',
    'label'         => 'Label',
    'name'          => '{prefix}_{name}',
    'type'          => 'select',
    'choices'       => array( 'option1' => 'Display 1', 'option2' => 'Display 2' ),
    'default_value' => 'option1',
),
```

**Repeater:**
```php
array(
    'key'          => 'field_{prefix}_{name}',
    'label'        => 'Label',
    'name'         => '{prefix}_{name}',
    'type'         => 'repeater',
    'layout'       => 'block',       // or 'table', 'row'
    'button_label' => 'Add Item',
    'sub_fields'   => array(
        array( 'key' => 'field_{prefix}_{sub}', 'label' => 'Sub Label', 'name' => 'sub_name', 'type' => 'text' ),
    ),
),
```

**WYSIWYG (basic, no media):**
```php
array(
    'key'          => 'field_{prefix}_{name}',
    'label'        => 'Label',
    'name'         => '{prefix}_{name}',
    'type'         => 'wysiwyg',
    'toolbar'      => 'basic',
    'media_upload' => 0,
    'tabs'         => 'visual',
    'default_value'=> '<a href="#">Link 1</a> <a href="#">Link 2</a>',
),
```

## Repeater layout choices

- `'table'` — compact, good for simple sub-fields (stats: value + suffix + label)
- `'row'` — medium, sub-fields stacked per row (features: icon + title + desc)
- `'block'` — full width, good for complex sub-fields (products with image + multiple text fields)

## Key rules

1. **Always set `default_value`** for text fields — use the original hardcoded content
2. **Prefix all keys and names** to avoid collisions with other field groups
3. **Use `return_format => 'url'`** for images used in `src=""` or `background` CSS
4. **Use `return_format => 'id'`** only when you need `wp_get_attachment_image()` with responsive srcset
5. **One field group per template** — keeps admin UI organized with tabs
