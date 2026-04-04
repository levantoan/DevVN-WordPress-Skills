# Report Format — WordPress Security & Code Quality Audit

All reports MUST be in **Vietnamese**. The audit rules and skill instructions are in English, but output to the user is always in Vietnamese.

## Per-issue format

```
[ISSUE_TYPE] - [Severity: Critical/High/Medium/Low]
File: path/to/file.php (Line: 123)
Code: `problematic code snippet`
Reason: Why this is a vulnerability (output in Vietnamese)
Fix: How to fix it (output in Vietnamese)
```

## Issue types — Security

| Code | Description |
|------|-------------|
| `SQLi` | SQL Injection — missing `$wpdb->prepare()` or improper escaping |
| `DIRECT_DB` | Direct database query — should use WP abstraction layer |
| `XSS` | Cross-Site Scripting — missing output escaping |
| `LATE_ESCAPE` | Late escaping violation — escaping too early or not at output |
| `INPUT_VALIDATION` | Missing input validation/sanitization on superglobals |
| `CSRF` | Cross-Site Request Forgery — missing nonce |
| `NONCE_BUG` | Buggy nonce verification pattern (unsafe if/else/negation logic) |
| `ACCESS_CONTROL` | Missing capability/permission check |
| `FILE_UPLOAD` | Unvalidated file upload |
| `UNFILTERED_UPLOAD` | `ALLOW_UNFILTERED_UPLOADS` constant found |
| `FILE_WRITE` | Unsafe file write — missing `WP_Filesystem` |
| `PATH_TRAVERSAL` | User-supplied path without `validate_file()` |
| `INSECURE_DESERIALIZATION` | `unserialize()` on untrusted data |
| `SSL_DISABLED` | SSL certificate verification disabled |
| `INFO_DISCLOSURE` | Sensitive data leaked (keys, tokens, debug output) |
| `NONCE_CONFLICT` | Multiple handlers matching same URL params |
| `UNSAFE_REDIRECT` | `wp_redirect()` used instead of `wp_safe_redirect()` |
| `MISSING_EXIT` | Missing `exit`/`die` after redirect |
| `FORBIDDEN_FUNCTION` | Forbidden function used (`eval`, `create_function`, etc.) |
| `DEV_FUNCTION` | Development function in production (`var_dump`, `phpinfo`, etc.) |
| `CODE_OBFUSCATION` | Code obfuscation detected (Zend, ionCube, SourceGuardian) |
| `ERROR_SUPPRESSION` | `@` operator suppressing errors |

## Issue types — Code Quality (for WordPress.org submission)

| Code | Description |
|------|-------------|
| `PREFIXING` | Global function/class/constant/variable missing plugin prefix |
| `SETTING_SANITIZE` | `register_setting()` missing `sanitize_callback` |
| `ENQUEUE` | Script/style not properly enqueued via `wp_enqueue_*` |
| `SCRIPT_SCOPE` | Script/style loaded on all pages instead of conditionally |
| `SCRIPT_STRATEGY` | Script missing loading strategy (`defer`/`async`/footer) |
| `ASSET_SIZE` | Enqueued asset size exceeds threshold |
| `SLOW_QUERY` | WP_Query with slow parameters (`posts_per_page => -1`, etc.) |
| `OFFLOADING` | External CDN resource — should be bundled locally |
| `DEPRECATED` | Deprecated WordPress function/class/parameter |
| `I18N` | Internationalization issue (wrong text domain, missing comments) |
| `LOCALHOST` | Hardcoded localhost/127.0.0.1 reference |
| `FILE_TYPE` | Forbidden file type in plugin directory |
| `PHP_SYNTAX` | PHP syntax issue (short open tags, BOM, etc.) |
| `PLUGIN_HEADER` | Missing/invalid plugin header field |
| `UNINSTALL` | Missing proper uninstall handling |
| `CUSTOM_UPDATER` | Custom update checker in WordPress.org plugin |
| `EXTERNAL_MENU` | External URL in admin menu |

## Severity levels

| Level | Description | Examples |
|-------|-------------|---------|
| **Critical** | RCE, direct SQLi, remote file write, arbitrary code execution | `eval()` on user input, `unserialize()` on remote data, `file_put_contents()` from user data, `ALLOW_UNFILTERED_UPLOADS` |
| **High** | Stored XSS, CSRF on dangerous actions, auth bypass, open redirect | Raw `echo $_GET`, missing nonce on delete handler, `wp_redirect()` with user-supplied URL, buggy nonce pattern |
| **Medium** | Limited reflected XSS, missing escape on DB values, missing input validation | `echo $row->name` without `esc_html()`, superglobal without `isset()`, missing `sanitize_callback` |
| **Low** | Coding rule violations, style/performance issues, not directly exploitable | Uncast numeric value, missing prefix, script in header without defer, localhost reference |

## Summary table (end of report)

```markdown
| # | File | Line(s) | Type | Severity | Exploitable |
|---|------|---------|------|----------|-------------|
| 1 | src/Controller.php | 45 | SQLi | Critical | Yes |
| 2 | templates/page.php | 89 | XSS | Medium | No (int value) |
| 3 | includes/Settings.php | 23 | SETTING_SANITIZE | Medium | No |
| 4 | admin/handler.php | 67 | NONCE_BUG | High | Yes |
```

## "PASS" section (end of report)

List all categories that were audited and found clean:
- SQL Injection: `$wpdb->prepare()` used correctly throughout
- XSS: All output properly escaped
- CSRF: All handlers verify nonce
- Access Control: `current_user_can()` before every action
- File Security: `WP_Filesystem` used, uploads validated
- Safe Redirect: `wp_safe_redirect()` used correctly
- Forbidden Functions: No dangerous functions found
- Input Validation: All inputs validated and sanitized
- Prefixing: All globals properly prefixed (if checked)
- Performance: Scripts/styles properly scoped (if checked)
- i18n: Text domain correct throughout (if checked)

This helps the user understand the audit was thorough, not just error-focused.

## Files to scan

```
# Root bootstrap
*.php

# Source (PSR-4)
src/**/*.php

# Templates (XSS hotspot)
templates/**/*.php

# Legacy includes
includes/**/*.php

# Admin views
admin/**/*.php

# JavaScript receiving PHP data
assets/**/*.js (check wp_localize_script, wp_add_inline_script)

# Block assets
blocks/**/*.php
blocks/**/*.js
```

## Agent prompt template

When using the Agent tool to run the audit, use this prompt structure:

```
Perform a COMPLETE WordPress security audit on this plugin.
Follow ALL rules in the audit-rules reference.
Scan EVERY PHP file in: [list directories]
Report using the exact format in report-format reference.
Report in Vietnamese.
ONLY report confirmed issues. NO false positives.
Do NOT report intelephense IDE warnings.
Report duplicate patterns once with "same pattern in N files".
Check for buggy nonce patterns (unsafe if/else/negation).
Check for wp_redirect vs wp_safe_redirect.
Check for forbidden functions (eval, create_function, backticks).
Check for missing input validation on superglobals.
```

For full audit (WordPress.org submission), add:
```
Also check code quality rules from code-quality-rules reference:
- Prefixing (all globals must have unique prefix)
- Setting sanitization (register_setting with sanitize_callback)
- Performance (script loading strategy, scoped loading, asset size)
- i18n (text domain, translator comments)
- Localhost references
- Deprecated functions
- Plugin headers and uninstall handling
```
