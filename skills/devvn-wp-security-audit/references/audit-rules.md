# Audit Rules — WordPress Plugin Security (WPCS)

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

### 2.2 Output Escaping
All data output to HTML must be escaped by context:

| Context | Function |
|---------|----------|
| HTML Body / Text content | `esc_html()` |
| HTML Attributes (value, title, alt) | `esc_attr()` |
| URLs (href, src, action) | `esc_url()` |
| Textarea content | `esc_textarea()` |
| JavaScript inline | `wp_json_encode()` or `esc_js()` |
| Translations | `esc_html__()`, `esc_attr_e()` |

### 2.3 Unsafe Output
Flag any `echo`, `print`, `printf` with variables not wrapped in `esc_*` functions. Even numeric DB values must be cast `(int)` or escaped.

```php
// WRONG
echo $row->count;
echo $user_input;

// CORRECT
echo (int) $row->count;
echo esc_html($user_input);
```

### 2.4 JavaScript Data
Data passed from PHP to JS via `wp_localize_script()` or `wp_add_inline_script()` must use `wp_json_encode()`.

```php
// CORRECT
wp_add_inline_script('handle', 'var CONFIG = ' . wp_json_encode($data) . ';', 'before');
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

### 4.3 Direct File Access
Every PHP file must include direct access protection at the top:

```php
defined('ABSPATH') || exit;
```

### 4.4 Multi-tenant Isolation
If the plugin has a multi-location/branch system:
- Every write action must verify that `location_id` belongs to the user's assigned locations
- Super admin (`manage_options`) bypasses this check

---

## 5. File & Remote Security

### 5.1 File Upload Validation
When accepting uploads, validate the file type:
- Basic: `wp_check_filetype($filename, $allowed_mimes)`
- Better: `wp_check_filetype_and_ext($tmp_name, $filename, $mimes)` (checks actual MIME content)

### 5.2 Filesystem API
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

### 5.3 Path Traversal
- Use `validate_file()` to block access to system directories via user-supplied paths
- Never allow user-supplied paths directly in `include`, `require`, `fopen`

### 5.4 Remote Data Deserialization
- **Never** use `unserialize()` on data from remote servers — use `json_decode()` or `unserialize($data, ['allowed_classes' => ['stdClass']])`
- **Never** suppress errors with `@` before deserialization functions
- **Never** disable SSL verification: `CURLOPT_SSL_VERIFYPEER` must be `true`

### 5.5 Sensitive Data Exposure
- License keys and API tokens must be sent in POST body, not in URL query strings
- Never echo secrets (keys, tokens) to frontend HTML or JavaScript
- Never log passwords or authentication tokens

---

## 6. Additional Patterns

### 6.1 die() / exit()
Only use immediately after `wp_safe_redirect()`. Never use elsewhere.

### 6.2 wp_safe_redirect()
Never call inside a page callback (headers already sent). Use `admin_init` hooks for redirects.

### 6.3 Error Suppression
Never use `@` before function calls — it silences error logs needed for debugging.
