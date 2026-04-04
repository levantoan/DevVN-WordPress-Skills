---
name: devvn-wp-security-audit
description: "Comprehensive WordPress plugin/theme security and code quality audit following WPCS and Plugin Check standards. Scans all PHP files for SQL Injection, XSS, CSRF, Access Control, File Security, Forbidden Functions, Safe Redirect, Input Validation, and Code Quality issues. Reports with file:line references and fix suggestions. Use when reviewing security before releasing a plugin, checking for vulnerabilities, preparing for WordPress.org submission, or running a code quality audit."
compatibility: "WordPress plugins and themes. Requires PHP files. Works best with plugins containing $wpdb queries, REST API endpoints, admin handlers, template files, and wp_enqueue_* calls."
metadata:
  author: devvn
  version: "2.0"
---

# DevVN WordPress Security & Code Quality Audit

Performs a full security and code quality audit on WordPress plugins/themes against WPCS (WordPress Coding Standards) and WordPress Plugin Check standards. Covers 8 vulnerability categories with 40+ individual rules, plus code quality and performance checks.

## When to use

Activate when:
- User asks to "check security", "security audit", "kiểm tra bảo mật", "audit plugin"
- User asks to "check code quality", "plugin check", "review code"
- Before publishing/releasing a plugin or submitting to WordPress.org
- User mentions "devvn-wp-security-audit"
- User asks "are there any security vulnerabilities" or "có lỗi bảo mật không"
- User asks about SQL injection, XSS, CSRF, nonce, escaping, sanitization

## Audit process

### Step 1: Identify scope

Scan **all** PHP files in the project:
- `src/**/*.php` (controllers, services, admin handlers)
- `templates/**/*.php` (highest XSS risk)
- `includes/**/*.php` (legacy logic)
- `admin/**/*.php` (admin views)
- Root `*.php` (bootstrap files)
- `assets/**/*.js` (check data passed from PHP via `wp_localize_script`, `wp_add_inline_script`)

Priority order: Controllers/Handlers > Templates > Services > Admin views > Bootstrap

### Step 2: Run security audit

Use the Agent tool with `subagent_type: feature-dev:code-reviewer`. Pass the **complete** audit rules from [references/audit-rules.md](references/audit-rules.md) as the prompt.

Instruct the agent to:
- Read ALL files completely before reporting
- Report using the exact format from [references/report-format.md](references/report-format.md)
- Only report confirmed issues, no false positives
- Skip intelephense IDE warnings (undefined function for WP globals is normal)
- Report duplicate patterns once, note "same pattern in N files"
- Check **current** code state, not previously fixed issues

### Step 3: Run code quality audit (if preparing for WordPress.org)

If the user is preparing for WordPress.org submission or asks for a full audit, also check rules from [references/code-quality-rules.md](references/code-quality-rules.md). This covers:
- Prefixing (all globals must have unique prefix)
- Setting sanitization (`register_setting()` with `sanitize_callback`)
- Performance (enqueued resources, WP_Query params, script loading strategy)
- i18n (text domain, translator comments)
- Localhost references
- Forbidden file types
- Plugin headers and uninstall handling

### Step 4: Report

Output results in Vietnamese using the format from [references/report-format.md](references/report-format.md). Include:
- Each issue with `[TYPE] - [Severity]`, file:line, code snippet, reason, fix
- Summary table at the end
- "PASS" section listing categories that were audited and found clean

### Step 5: Fix

Fix reported issues from Critical to Low, but **ask the user before fixing** patterns that may be intentional:
- SSL verification disabled in cURL fallback (may be needed for hosting compatibility)
- Sensitive data passed to frontend JS via `wp_localize_script()` (may be required by frontend logic)
- `wp_redirect()` usage (may be intentional for external redirects)
- Development functions like `error_log()` (may be needed for debugging)
- Other patterns where fixing could break existing functionality

Only auto-fix issues that are clearly unintentional (missing `esc_html()`, missing `wp_check_filetype()`, raw `unserialize()`, etc.). Run `php -l` on every modified file to verify syntax.

## Audit categories

### 1. SQL Injection (SQLi)
- `$wpdb->prepare()` required for all queries with user-supplied data
- Placeholder count must match parameter count
- `$wpdb->esc_like()` required for LIKE clauses
- `array_fill` for IN() placeholders, never string concatenation
- ORDER BY/GROUP BY must use whitelist validation, never `%s`
- Prefer WP abstraction (`get_posts`, `WP_Query`) over direct `$wpdb` when possible

### 2. Cross-Site Scripting (XSS)
- All `$_GET`/`$_POST`/`$_REQUEST` must be sanitized on input
- All output must be escaped **late** (right before echo): `esc_html()`, `esc_attr()`, `esc_url()`, `esc_textarea()`
- Flag any `echo`/`print`/`printf` of variables without `esc_*` functions
- Even numeric DB values must be cast `(int)` or escaped before output
- JS data from PHP must use `wp_json_encode()`
- Flag unsafe printing functions and unescaped search queries

### 3. CSRF (Nonce)
- Every action form/URL must have `wp_nonce_field()` or `wp_nonce_url()`
- Every handler must verify with `check_admin_referer()` or `wp_verify_nonce()`
- Flag any `$_POST` handler without nonce verification
- Detect nonce conflicts when multiple handlers match same URL params
- Detect buggy nonce patterns: unsafe if-statement logic, negated AND, problematic else

### 4. Access Control
- `current_user_can()` required before every sensitive action
- REST API write endpoints must have proper `permission_callback`
- Every PHP file must have `defined('ABSPATH') || exit` (or `WPINC`)
- Branch/multi-tenant isolation must verify ownership before writes

### 5. File & Remote Security
- File uploads must validate type with `wp_check_filetype()`
- Use `WP_Filesystem` API instead of raw `file_put_contents()`/`fopen()`
- Never use `unserialize()` on remote data
- Never disable SSL: `CURLOPT_SSL_VERIFYPEER` must be `true`
- Sensitive tokens/keys must not appear in URLs or frontend output
- Path traversal: `validate_file()` for user-supplied paths
- `ALLOW_UNFILTERED_UPLOADS` constant is forbidden

### 6. Safe Redirect
- Use `wp_safe_redirect()` instead of `wp_redirect()` for internal redirects
- Always call `exit` or `die` after redirect
- Never redirect inside page callbacks (headers already sent)

### 7. Forbidden & Dangerous Functions
- Never use `eval()`, `create_function()`, `assert()` with string args
- Never use `move_uploaded_file()` directly — use WP upload handling
- Never use backtick operator (`` ` ``) for shell execution
- Flag `set_time_limit()`, `ini_set()` in production code
- No code obfuscation (Zend Guard, ionCube, Source Guardian)
- No `goto` construct

### 8. Input Validation
- All superglobals must be validated AND sanitized before use
- Use `wp_unslash()` before sanitizing `$_POST`/`$_GET` values
- Never process the entire `$_POST`/`$_GET` array — read explicit keys
- `register_setting()` must have `sanitize_callback`

## Gotchas

Common false positives to **avoid reporting**:
- `esc_html__()` and `esc_attr_e()` are valid — they combine escaping + translation
- `wp_kses_post()` is valid sanitization for trusted HTML in admin context
- `absint()` is valid sanitization for positive integers (no need for additional `(int)` cast)
- `$wpdb->prepare()` with `{$prefix}` for table name is acceptable (table prefix, not user data)
- WordPress core functions like `wp_insert_post()`, `update_option()` handle their own sanitization
- `__return_true` as REST `permission_callback` is acceptable for public read endpoints

Common patterns that **look safe but aren't**:
- `intval($_GET['id'])` without `$wpdb->prepare()` — still needs prepare for SQL
- `esc_html()` on output but no `sanitize_*()` on input — both are needed
- Nonce checked but no `current_user_can()` — nonce prevents CSRF, not authorization
- `wp_verify_nonce()` in an `if` without early return on failure — code continues to execute

## Important notes

- **All reports must be in Vietnamese** — the audit rules and skill instructions are in English, but output to the user in Vietnamese
- Do NOT report false positives — only confirmed issues
- Do NOT report the same pattern multiple times — mention "same pattern in N files"
- Prioritize exploitable issues first, defensive coding issues second
- After fixing, verify with `php -l` syntax check on all modified files
- When in doubt about severity, check [references/report-format.md](references/report-format.md) for severity definitions
