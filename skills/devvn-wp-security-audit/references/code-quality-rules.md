# Code Quality & Compliance Rules — WordPress Plugin Check

These rules are based on the WordPress Plugin Check plugin and WordPress.org submission requirements. Apply when the user asks for a full audit or is preparing for WordPress.org submission.

---

## 1. Prefixing

### 1.1 Global Namespace Pollution
All functions, classes, constants, and global variables must use a unique plugin prefix to avoid conflicts.

```php
// WRONG — generic names pollute global namespace
function get_settings() {}
class Logger {}
define('VERSION', '1.0');
global $config;

// CORRECT — prefixed with plugin slug
function myplugin_get_settings() {}
class MyPlugin_Logger {}
define('MYPLUGIN_VERSION', '1.0');
global $myplugin_config;
```

### 1.2 Prefix Detection
The prefix should be derived from the plugin slug (text domain). Check that:
- All `function` declarations use the prefix
- All `class` declarations use the prefix (or namespace)
- All `define()` constants use the prefix
- All `global $` variables use the prefix
- Hook callback function names use the prefix

Exception: Methods inside a prefixed class don't need additional prefixing.

---

## 2. Enqueued Resources

### 2.1 Proper Enqueuing
All scripts and styles must be enqueued using WordPress functions, never hardcoded in HTML:

```php
// WRONG — hardcoded in template
echo '<script src="plugin/js/app.js"></script>';
echo '<link rel="stylesheet" href="plugin/css/style.css">';

// CORRECT
wp_enqueue_script('myplugin-app', plugins_url('js/app.js', __FILE__), [], '1.0', true);
wp_enqueue_style('myplugin-style', plugins_url('css/style.css', __FILE__), [], '1.0');
```

### 2.2 Script Loading Strategy
Scripts should specify a loading strategy for better performance:

```php
// GOOD — explicit footer loading
wp_enqueue_script('handle', $url, [], '1.0', true);

// BETTER — defer strategy (WP 6.3+)
wp_enqueue_script('handle', $url, [], '1.0', ['strategy' => 'defer']);
```

### 2.3 Scoped Loading
Scripts and styles should only be loaded on pages where they are needed:

```php
// WRONG — loaded on every page
add_action('wp_enqueue_scripts', function() {
    wp_enqueue_script('myplugin-admin', ...);
});

// CORRECT — only on plugin's admin page
add_action('admin_enqueue_scripts', function($hook) {
    if ('toplevel_page_my-plugin' !== $hook) return;
    wp_enqueue_script('myplugin-admin', ...);
});
```

### 2.4 Asset Size
- Total enqueued JS should be under 300KB per page
- Total enqueued CSS should be under 250KB per page
- Flag excessively large bundles

---

## 3. Performance

### 3.1 WP_Query Parameters
Flag slow WP_Query parameters:

```php
// SLOW — avoid in production
new WP_Query([
    'meta_query' => [...],      // Slow without proper indexing
    'tax_query' => [...],       // Can be slow with complex queries
    'posts_per_page' => -1,     // Loads ALL posts — never do this
]);

// BETTER
new WP_Query([
    'posts_per_page' => 100,    // Always set a reasonable limit
    'no_found_rows' => true,    // Skip SQL_CALC_FOUND_ROWS if no pagination
    'update_post_meta_cache' => false, // Skip if not using meta
    'update_post_term_cache' => false, // Skip if not using terms
]);
```

### 3.2 Offloading Files
Don't load scripts/styles from external CDNs — bundle them locally:

```php
// WRONG — depends on external service
wp_enqueue_script('jquery-cdn', 'https://cdn.jsdelivr.net/npm/jquery/dist/jquery.min.js');

// CORRECT — use local copy or WordPress bundled version
wp_enqueue_script('jquery');
```

### 3.3 Deprecated WordPress Functions
Flag usage of deprecated WordPress functions, classes, constants, and parameters. Common examples:
- `get_bloginfo('url')` → use `home_url()`
- `get_bloginfo('wpurl')` → use `site_url()`
- `query_posts()` → use `WP_Query` or `get_posts()`
- `get_page_by_title()` → use `WP_Query` with `'title'` parameter

---

## 4. Internationalization (i18n)

### 4.1 Text Domain
All translatable strings must use the correct text domain matching the plugin slug:

```php
// WRONG — wrong or missing text domain
__('Hello World');
__('Hello World', 'wrong-domain');

// CORRECT — matches plugin slug
__('Hello World', 'my-plugin');
```

### 4.2 Translator Comments
Strings with placeholders must include translator comments:

```php
// WRONG — no context for translators
printf(__('%s items found', 'my-plugin'), $count);

// CORRECT
/* translators: %s: number of items */
printf(__('%s items found', 'my-plugin'), $count);
```

### 4.3 i18n Anti-patterns
- Never use variables as text domain: `__('text', $domain)`
- Never use variables as translatable string: `__($variable, 'domain')`
- Never pass empty strings to i18n functions
- Never use restricted text domains: `textdomain`, `text-domain`, `text_domain`, `your-text-domain`
- Use ordered placeholders for multiple: `%1$s`, `%2$s` (not `%s`, `%s`)

---

## 5. Plugin Structure

### 5.1 Plugin Headers
The main plugin file must have all required headers:

```php
/**
 * Plugin Name: My Plugin
 * Description: What the plugin does.
 * Version: 1.0.0
 * Author: Author Name
 * License: GPL-2.0-or-later
 * Text Domain: my-plugin
 * Requires at least: 6.0
 * Requires PHP: 7.4
 */
```

### 5.2 Uninstall Handling
Plugins must clean up data on uninstall using one of:
- `uninstall.php` file with `defined('WP_UNINSTALL_PLUGIN') || exit;`
- `register_uninstall_hook()` in main plugin file

### 5.3 No Custom Updater
WordPress.org plugins must NOT include custom update checkers. Flag:
- `plugin-update-checker.php` files
- `WP_GitHub_Updater` class
- `PucFactory::buildUpdateChecker` calls
- Hooks on `pre_set_site_transient_update_plugins` or `auto_update_plugin`

Exception: This rule does NOT apply to plugins distributed outside WordPress.org (e.g., via license server).

---

## 6. Forbidden Content

### 6.1 Localhost References
Never hardcode `localhost` or `127.0.0.1` in production code:

```php
// WRONG
$api_url = 'http://localhost:8080/api';
$db_host = '127.0.0.1';

// CORRECT — use configuration or constants
$api_url = get_option('myplugin_api_url');
```

### 6.2 Forbidden File Types
Plugins must NOT contain:
- Compressed files: `.zip`, `.gz`, `.tgz`, `.rar`, `.tar`, `.7z`
- PHAR archives: `.phar`
- VCS directories: `.git`, `.svn`, `.hg`
- Binary executables: `.exe`, `.so`, `.bin`, `.dmg`
- AI instruction directories: `.cursor`, `.claude`, `.aider`, `.windsurf`

### 6.3 PHP Syntax Issues
- No short open tags: `<?` must be `<?php`
- No alternative PHP tags: `<%`, `<script language="php">`
- No heredoc/nowdoc in class properties (PHP < 7.3 compat issue)
- No BOM (Byte Order Mark) in PHP files

### 6.4 External Admin Menu Links
Top-level admin menu items must NOT link to external URLs:

```php
// WRONG — external URL as menu slug
add_menu_page('Plugin', 'Plugin', 'manage_options', 'https://example.com/upgrade');

// CORRECT
add_menu_page('Plugin', 'Plugin', 'manage_options', 'my-plugin-settings', 'callback');
```

### 6.5 5-Star Review Links
Do not include links to filtered 5-star reviews on WordPress.org:

```php
// FORBIDDEN
$url = 'https://wordpress.org/support/plugin/my-plugin/reviews/?filter=5';

// ACCEPTABLE
$url = 'https://wordpress.org/support/plugin/my-plugin/reviews/';
```
