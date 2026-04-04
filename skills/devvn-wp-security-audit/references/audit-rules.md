# Audit Rules — WordPress Plugin Security (WPCS + Plugin Check)

## 1. SQL Injection (SQLi)

### 1.1 Prepare Rule
Every SQL statement containing variables (user-supplied data) MUST use `$wpdb->prepare()`.

```php
// WRONG — direct concatenation
$wpdb->get_results("SELECT * FROM $table WHERE id = " . $_GET['id']);

// CORRECT
$wpdb->get_results($wpdb->prepare("SELECT * FROM $table WHERE id = %d", absint($_GET['id'])));
```

### 1.2 Placeholder Match
The number of placeholders (`%s`, `%d`, `%f`) must exactly match the number of parameters passed.

### 1.3 LIKE Clause
User data in LIKE clauses must pass through `$wpdb->esc_like()` before `prepare()`.

```php
// CORRECT
$like = $wpdb->esc_like($search) . '%';
$wpdb->prepare("SELECT * FROM $table WHERE name LIKE %s", $like);
```

### 1.4 IN Clause
Never concatenate strings for `IN()`. Use `array_fill` to generate placeholders and spread the array into `prepare()`.

```php
// WRONG
$ids_str = implode(',', $ids);
$wpdb->get_results("SELECT * FROM $table WHERE id IN ($ids_str)");

// CORRECT
$placeholders = implode(',', array_fill(0, count($ids), '%d'));
$wpdb->get_results($wpdb->prepare(
    "SELECT * FROM $table WHERE id IN ({$placeholders})",
    ...$ids
));
```

### 1.5 Dynamic Ordering
ORDER BY and GROUP BY parameters cannot use `%s`. Must validate against a **whitelist**.

```php
// CORRECT
$valid_columns = ['id', 'name', 'created_at'];
$orderby = in_array($orderby, $valid_columns, true) ? $orderby : 'id';
$order = 'ASC' === strtoupper($order) ? 'ASC' : 'DESC';
```

### 1.6 Prefer WP Abstraction
When possible, use WordPress query functions (`get_posts()`, `WP_Query`, `get_option()`, `get_user_meta()`) instead of direct `$wpdb` queries. Direct `$wpdb` should only be used for complex queries, custom tables, or performance-critical operations.

```php
// DISCOURAGED — direct DB query for standard data
$wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_type = 'product'");

// PREFERRED — WP abstraction handles escaping and caching
$products = get_posts(['post_type' => 'product']);
```

### 1.7 Restricted DB Classes
Never use `mysqli_*`, `mysql_*`, or raw PDO for database access. Always use `$wpdb`.

---

## 2. Cross-Site Scripting (XSS)

### 2.1 Input Sanitization
All data from `$_GET`, `$_POST`, `$_REQUEST` must be sanitized immediately on receipt:

| Function | Use for |
|----------|---------|
| `sanitize_text_field()` | Plain text |
| `sanitize_email()` | Email addresses |
| `absint()` | Positive integers |
| `sanitize_textarea_field()` | Multi-line text |
| `sanitize_title()` | Slugs |
| `esc_url_raw()` | URLs stored in DB |
| `wp_kses_post()` | Trusted HTML (admin only) |
| `sanitize_key()` | Keys, slugs, lowercase alphanumeric |
| `sanitize_file_name()` | File names |
| `sanitize_mime_type()` | MIME types |

### 2.2 Output Escaping (Late Escaping)
All data output to HTML must be escaped **right before output** (late escaping), never early:

| Context | Function |
|---------|----------|
| HTML Body / Text content | `esc_html()` |
| HTML Attributes (value, title, alt) | `esc_attr()` |
| URLs (href, src, action) | `esc_url()` |
| Textarea content | `esc_textarea()` |
| JavaScript inline | `wp_json_encode()` or `esc_js()` |
| Translations | `esc_html__()`, `esc_attr_e()`, `esc_html_e()` |
| CSS | `safecss_filter_attr()` |

### 2.3 Unsafe Output
Flag any `echo`, `print`, `printf`, `vprintf` with variables not wrapped in `esc_*` functions. Even numeric DB values must be cast `(int)` or escaped.

```php
// WRONG
echo $row->count;
echo $user_input;
printf('<a href="%s">%s</a>', $url, $title);

// CORRECT
echo (int) $row->count;
echo esc_html($user_input);
printf('<a href="%s">%s</a>', esc_url($url), esc_html($title));
```

### 2.4 Unsafe Printing Functions
Flag use of `print_r()`, `var_dump()`, `var_export()` with output to browser (without `true` parameter or outside of logging context).

### 2.5 JavaScript Data
Data passed from PHP to JS via `wp_localize_script()` or `wp_add_inline_script()` must use `wp_json_encode()`.

```php
// CORRECT
wp_add_inline_script('handle', 'var CONFIG = ' . wp_json_encode($data) . ';', 'before');
```

### 2.6 Search Query Escaping
When displaying search queries back to the user, always escape:

```php
// WRONG
echo 'Search results for: ' . get_search_query();

// CORRECT
echo 'Search results for: ' . get_search_query(true); // true = escaped
// OR
echo 'Search results for: ' . esc_html(get_search_query(false));
```

### 2.7 Input Validation Before Use
All superglobals (`$_GET`, `$_POST`, `$_REQUEST`, `$_SERVER`, `$_COOKIE`) must be validated with `isset()` or `array_key_exists()` AND sanitized before use. Never access superglobals without checking existence first.

```php
// WRONG
$page = $_GET['page'];

// CORRECT
$page = isset($_GET['page']) ? sanitize_text_field(wp_unslash($_GET['page'])) : '';
```

### 2.8 wp_unslash Before Sanitize
Always use `wp_unslash()` before sanitizing `$_POST`/`$_GET`/`$_REQUEST` values, because WordPress adds magic quotes.

```php
// WRONG
$name = sanitize_text_field($_POST['name']);

// CORRECT
$name = sanitize_text_field(wp_unslash($_POST['name']));
```

---

## 3. CSRF Protection (Nonce)

### 3.1 Nonce Generation
Every form or URL that performs an action (delete, save, update) must include:
- Form: `wp_nonce_field('action_name')`
- URL: `wp_nonce_url($url, 'action_name')`
- AJAX: `wp_create_nonce('action_name')`

### 3.2 Nonce Verification
Every entry point must verify the nonce:
- Admin form: `check_admin_referer('action_name')`
- AJAX: `check_ajax_referer('action_name')`
- REST API: `wp_verify_nonce()` in `permission_callback`
- Public form: `wp_verify_nonce()` (do NOT use `check_admin_referer` for public pages)

### 3.3 Missing Nonce
Flag any `$_POST` handler that does not check `_wpnonce`.

### 3.4 Nonce Action Conflicts
When multiple handlers match the same URL parameters (e.g., `page=X&action=delete`), each handler must include a tab/scope guard to prevent nonce mismatch.

```php
// When a page has tabs, the default-tab handler must reject requests with a tab param
if (!empty($_GET['tab'])) { return; }
```

### 3.5 Buggy Nonce Patterns
Detect these common bugs in nonce verification (from Plugin Check `VerifyNonce` sniff):

**Unsafe if-statement:** `wp_verify_nonce()` used in a compound `if` where execution continues even when nonce fails:
```php
// WRONG — code after if still executes on nonce failure
if (wp_verify_nonce($_POST['_wpnonce'], 'action')) {
    // safe path
}
// dangerous code here still runs!

// CORRECT — early return on failure
if (!wp_verify_nonce($_POST['_wpnonce'], 'action')) {
    wp_die('Security check failed');
}
```

**Negated AND:** Incorrect negation logic combining nonce with other checks:
```php
// WRONG — if nonce fails but other condition is true, code continues
if (!wp_verify_nonce($nonce, 'action') && !current_user_can('manage_options')) {
    wp_die('Unauthorized');
}

// CORRECT — check nonce independently
if (!wp_verify_nonce($nonce, 'action')) {
    wp_die('Nonce failed');
}
if (!current_user_can('manage_options')) {
    wp_die('Unauthorized');
}
```

**Problematic else:** Nonce verified in `if` but action performed in `else`:
```php
// WRONG — action in else means it runs when nonce FAILS
if (wp_verify_nonce($nonce, 'action')) {
    // log success
} else {
    process_form_data(); // BUG: runs on nonce failure!
}
```

---

## 4. Access Control

### 4.1 Capability Check
Every sensitive action must call `current_user_can()` before execution:
- Settings/management: `manage_options`
- Content: `edit_posts`
- Custom capabilities: e.g., `wbb_manage_own_location` (branch admin)

### 4.2 REST API
- Write endpoints (`POST`, `PUT`, `PATCH`, `DELETE`): `permission_callback` must verify nonce or capability
- Read endpoints: `__return_true` is acceptable for public data
- Apply rate limiting to endpoints vulnerable to abuse (enumeration, brute-force)
- NEVER use `permission_callback => ''` — always provide a callable

### 4.3 Direct File Access
Every PHP file must include direct access protection at the top. Accepted patterns:

```php
// Any of these are acceptable:
defined('ABSPATH') || exit;
defined('ABSPATH') || die;
defined('ABSPATH') || die();
defined('WPINC') || exit;
if (!defined('ABSPATH')) { exit; }
if (!defined('ABSPATH')) { die(); }
if (!defined('WPINC')) { exit; }
```

Exception: Files that only contain a class definition (no executable code) may omit this, though it is still recommended.

### 4.4 Multi-tenant Isolation
If the plugin has a multi-location/branch system:
- Every write action must verify that `location_id` belongs to the user's assigned locations
- Super admin (`manage_options`) bypasses this check

### 4.5 Menu Slug Security
Admin menu slugs should not expose internal file paths. Use abstract slugs instead:

```php
// WRONG — exposes file path
add_menu_page('Plugin', 'Plugin', 'manage_options', __FILE__, 'callback');

// CORRECT
add_menu_page('Plugin', 'Plugin', 'manage_options', 'my-plugin', 'callback');
```

---

## 5. File & Remote Security

### 5.1 File Upload Validation
When accepting uploads, validate the file type:
- Basic: `wp_check_filetype($filename, $allowed_mimes)`
- Better: `wp_check_filetype_and_ext($tmp_name, $filename, $mimes)` (checks actual MIME content)

### 5.2 Unfiltered Uploads
The `ALLOW_UNFILTERED_UPLOADS` constant must NEVER appear in plugin code. This allows uploading any file type including PHP executables.

```php
// FORBIDDEN — never define this in a plugin
define('ALLOW_UNFILTERED_UPLOADS', true);
```

### 5.3 Filesystem API
Do NOT use raw PHP functions for file operations:
- **Wrong:** `file_put_contents()`, `fwrite()`, `fopen()` for writing
- **Correct:** `WP_Filesystem` API

```php
global $wp_filesystem;
if (!function_exists('WP_Filesystem')) {
    require_once ABSPATH . 'wp-admin/includes/file.php';
}
WP_Filesystem();
$wp_filesystem->put_contents($file, $content, FS_CHMOD_FILE);
```

### 5.4 Path Traversal
- Use `validate_file()` to block access to system directories via user-supplied paths
- Never allow user-supplied paths directly in `include`, `require`, `fopen`
- Validate that paths stay within expected directories

```php
// WRONG
include $_GET['template'];

// CORRECT
$template = sanitize_file_name($_GET['template']);
if (0 !== validate_file($template)) {
    wp_die('Invalid file');
}
include plugin_dir_path(__FILE__) . 'templates/' . $template;
```

### 5.5 Remote Data Deserialization
- **Never** use `unserialize()` on data from remote servers — use `json_decode()` or `unserialize($data, ['allowed_classes' => ['stdClass']])`
- **Never** suppress errors with `@` before deserialization functions
- **Never** disable SSL verification: `CURLOPT_SSL_VERIFYPEER` must be `true`

### 5.6 Sensitive Data Exposure
- License keys and API tokens must be sent in POST body, not in URL query strings
- Never echo secrets (keys, tokens) to frontend HTML or JavaScript
- Never log passwords or authentication tokens

---

## 6. Safe Redirect

### 6.1 Use wp_safe_redirect
Always use `wp_safe_redirect()` instead of `wp_redirect()` for internal redirects. `wp_safe_redirect()` validates the destination against an allow-list of hosts, preventing open redirect vulnerabilities.

```php
// WRONG — allows redirect to any URL
wp_redirect($url);
exit;

// CORRECT — validates destination host
wp_safe_redirect($url);
exit;
```

### 6.2 Exit After Redirect
Always call `exit` or `die` immediately after any redirect function. Without it, PHP continues executing code after sending the redirect header.

```php
// WRONG — code continues after redirect
wp_safe_redirect(admin_url());
do_something_dangerous(); // This still executes!

// CORRECT
wp_safe_redirect(admin_url());
exit;
```

### 6.3 No Redirect in Page Callbacks
Never call redirect functions inside admin page callbacks or template rendering functions — headers have already been sent. Use `admin_init` or similar early hooks for redirects.

```php
// WRONG — inside page callback
function my_admin_page() {
    if ($condition) {
        wp_safe_redirect(admin_url()); // Headers already sent!
        exit;
    }
    echo '<div>...</div>';
}

// CORRECT — redirect from early hook
add_action('admin_init', function() {
    if ($condition) {
        wp_safe_redirect(admin_url());
        exit;
    }
});
```

---

## 7. Forbidden & Dangerous Functions

### 7.1 Code Execution Functions
These functions allow arbitrary code execution and must NEVER be used:

| Function | Risk |
|----------|------|
| `eval()` | Arbitrary PHP execution |
| `create_function()` | Deprecated, uses eval internally |
| `assert()` with string arg | Code execution in older PHP |
| Backtick operator (`` `command` ``) | Shell execution |
| `preg_replace()` with `/e` modifier | Code execution (deprecated) |

### 7.2 File Manipulation
These functions bypass WordPress filesystem handling:

| Function | Use Instead |
|----------|-------------|
| `move_uploaded_file()` | `wp_handle_upload()` |
| `file_put_contents()` | `$wp_filesystem->put_contents()` |
| `fwrite()` / `fopen()` for writing | `$wp_filesystem->put_contents()` |

### 7.3 Development/Debug Functions in Production
Flag these if found outside of `WP_DEBUG` guards:

| Function | Issue |
|----------|-------|
| `var_dump()` | Debug output to browser |
| `print_r()` (without `true`) | Debug output to browser |
| `var_export()` (without `true`) | Debug output to browser |
| `phpinfo()` | Information disclosure |
| `debug_backtrace()` / `debug_print_backtrace()` | Information disclosure |

Note: `error_log()` is acceptable for debugging but should be reviewed.

### 7.4 PHP Configuration
These should be flagged as warnings (may be needed but risky):

| Function | Risk |
|----------|------|
| `set_time_limit()` | Can mask performance issues |
| `ini_set()` | Changes PHP config at runtime |
| `putenv()` | Modifies environment variables |

### 7.5 Code Obfuscation
Code obfuscation is not permitted (violates GPL and WordPress.org guidelines). Detect:
- Zend Guard: `<?php @Zend;` or `This file was encoded by`
- Source Guardian: `sourceguardian.com`, `function_exists('sg_load')`, `$__x=`
- ionCube: `ionCube` references

### 7.6 Goto Construct
The `goto` construct should not be used — it makes code flow unpredictable and hard to audit.

### 7.7 Error Suppression
Never use `@` before function calls — it silences error logs needed for debugging. Exception: `@unlink()` for temporary file cleanup where failure is acceptable.

---

## 8. Input Validation

### 8.1 Explicit Key Access
Never process the entire `$_POST` or `$_GET` array. Always access specific keys:

```php
// WRONG
foreach ($_POST as $key => $value) {
    update_option($key, $value);
}

// CORRECT
$name = isset($_POST['name']) ? sanitize_text_field(wp_unslash($_POST['name'])) : '';
```

### 8.2 Setting Sanitization
Every `register_setting()` call must include a `sanitize_callback`:

```php
// WRONG — no sanitization
register_setting('my_group', 'my_option');

// CORRECT
register_setting('my_group', 'my_option', [
    'sanitize_callback' => 'sanitize_text_field',
]);
```

### 8.3 Type Validation
Validate data types before use:

```php
// Integers
$id = absint($_GET['id']);

// Arrays
if (!is_array($items)) { return; }

// Email
if (!is_email($email)) { return; }

// URLs
$url = esc_url_raw($url);
if (empty($url)) { return; }
```

### 8.4 Nonce Before Input Processing
Always verify the nonce BEFORE processing any input data:

```php
// WRONG — processes input before nonce check
$data = sanitize_text_field($_POST['data']);
check_admin_referer('save_action');
update_option('key', $data);

// CORRECT — nonce first, then process
check_admin_referer('save_action');
$data = sanitize_text_field(wp_unslash($_POST['data']));
update_option('key', $data);
```
