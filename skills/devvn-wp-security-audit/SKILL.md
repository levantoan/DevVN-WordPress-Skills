---
name: devvn-wp-security-audit
description: "Comprehensive WordPress plugin security audit following WPCS standards. Scans all PHP files for SQL Injection, XSS, CSRF, Access Control, and File Upload vulnerabilities. Reports with file:line references and fix suggestions. Use when reviewing security before releasing a plugin, or when asked to check for vulnerabilities."
compatibility: "WordPress plugins and themes. Requires PHP files with $wpdb queries, REST API endpoints, admin handlers, and template files."
metadata:
  author: devvn
  version: "1.0"
---

# DevVN WordPress Security Audit

Performs a full security audit on WordPress plugins/themes against WPCS (WordPress Coding Standards). Covers 5 vulnerability categories with 25+ individual rules.

## When to use

Activate when:
- User asks to "check security", "security audit", "check SQL injection", "check XSS"
- Before publishing/releasing a plugin
- User mentions "devvn-wp-security-audit" or asks in Vietnamese about security
- User asks "are there any security vulnerabilities"

## Audit process

### Step 1: Identify scope

Scan **all** PHP files in the project:
- `src/**/*.php` (controllers, services, admin handlers)
- `templates/**/*.php` (highest XSS risk)
- `includes/**/*.php` (legacy logic)
- Root `*.php` (bootstrap files)
- `assets/**/*.js` (check data passed from PHP via `wp_localize_script`, `wp_add_inline_script`)

Priority order: Controllers > Services > Admin handlers > Templates

### Step 2: Run audit

Use the Agent tool with `subagent_type: feature-dev:code-reviewer`. Pass the **complete** audit rules from [references/audit-rules.md](references/audit-rules.md) as the prompt.

Instruct the agent to:
- Read ALL files completely before reporting
- Report using the exact format from [references/report-format.md](references/report-format.md)
- Only report confirmed issues, no false positives
- Skip intelephense IDE warnings (undefined function for WP globals is normal)
- Report duplicate patterns once, note "same pattern in N files"
- Check **current** code state, not previously fixed issues

### Step 3: Report

Output results in Vietnamese using the format from [references/report-format.md](references/report-format.md). Include:
- Each issue with `[TYPE] - [Severity]`, file:line, code snippet, reason, fix
- Summary table at the end
- "PASS" section listing categories that were audited and found clean

### Step 4: Fix

Fix all reported issues from Critical to Low. Run `php -l` on every modified file to verify syntax.

## Audit categories

### 1. SQL Injection (SQLi)
- `$wpdb->prepare()` required for all queries with user-supplied data
- Placeholder count must match parameter count
- `$wpdb->esc_like()` required for LIKE clauses
- `array_fill` for IN() placeholders, never string concatenation
- ORDER BY/GROUP BY must use whitelist validation, never `%s`

### 2. Cross-Site Scripting (XSS)
- All `$_GET`/`$_POST`/`$_REQUEST` must be sanitized on input
- All output must be escaped by context: `esc_html()`, `esc_attr()`, `esc_url()`, `esc_textarea()`
- Flag any `echo`/`print`/`printf` of variables without `esc_*` functions
- Even numeric DB values must be cast `(int)` or escaped before output
- JS data from PHP must use `wp_json_encode()`

### 3. CSRF (Nonce)
- Every action form/URL must have `wp_nonce_field()` or `wp_nonce_url()`
- Every handler must verify with `check_admin_referer()` or `wp_verify_nonce()`
- Flag any `$_POST` handler without nonce verification
- Detect nonce conflicts when multiple handlers match same URL params

### 4. Access Control
- `current_user_can()` required before every sensitive action
- REST API write endpoints must have proper `permission_callback`
- Every PHP file must have `defined('ABSPATH') || exit`
- Branch/multi-tenant isolation must verify ownership before writes

### 5. File & Remote Security
- File uploads must validate type with `wp_check_filetype()`
- Use `WP_Filesystem` API instead of raw `file_put_contents()`/`fopen()`
- Never use `unserialize()` on remote data — use `json_decode()` or `allowed_classes`
- Never disable SSL: `CURLOPT_SSL_VERIFYPEER` must be `true`
- Sensitive tokens/keys must not appear in URLs or frontend output

## Important notes

- **All reports must be in Vietnamese** — the audit rules and skill instructions are in English, but output to the user in Vietnamese
- Do NOT report false positives — only confirmed issues
- Do NOT report the same pattern multiple times — mention "same pattern in N files"
- Prioritize exploitable issues first, defensive coding issues second
- After fixing, verify with `php -l` syntax check on all modified files
