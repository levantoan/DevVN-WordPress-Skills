# Report Format — WordPress Security Audit

All reports MUST be in **Vietnamese**. The audit rules and skill instructions are in English, but output to the user is always in Vietnamese.

## Per-issue format

```
[ISSUE_TYPE] - [Severity: Critical/High/Medium/Low]
File: path/to/file.php (Line: 123)
Code: `problematic code snippet`
Reason: Why this is a vulnerability (output in Vietnamese)
Fix: How to fix it (output in Vietnamese)
```

## Issue types

| Code | Description |
|------|-------------|
| `SQLi` | SQL Injection |
| `XSS` | Cross-Site Scripting |
| `CSRF` | Cross-Site Request Forgery (missing nonce) |
| `ACCESS_CONTROL` | Missing capability/permission check |
| `FILE_UPLOAD` | Unvalidated file upload |
| `FILE_WRITE` | Unsafe file write (missing WP_Filesystem) |
| `PATH_TRAVERSAL` | User-supplied path without validation |
| `INSECURE_DESERIALIZATION` | unserialize() on untrusted data |
| `SSL_VERIFICATION_DISABLED` | SSL certificate verification disabled |
| `INFORMATION_DISCLOSURE` | Sensitive data leaked (keys, tokens) |
| `NONCE_CONFLICT` | Multiple handlers matching same URL params causing nonce mismatch |

## Severity levels

| Level | Description | Example |
|-------|-------------|---------|
| **Critical** | RCE, direct SQLi, remote file write | `unserialize()` on remote data, `file_put_contents()` from server |
| **High** | Stored XSS, CSRF on dangerous actions, auth bypass | Raw `echo $_GET`, missing nonce on delete handler |
| **Medium** | Limited reflected XSS, missing escape on DB values | `echo $row->name` without `esc_html()` |
| **Low** | Coding rule violations, not exploitable | Uncast numeric computed value |

## Summary table (end of report)

```markdown
| # | File | Line(s) | Type | Severity | Exploitable |
|---|------|---------|------|----------|-------------|
| 1 | src/Controller.php | 45 | SQLi | Critical | Yes |
| 2 | templates/page.php | 89 | XSS | Medium | No (int value) |
```

## "PASS" section (end of report)

List all categories that were audited and found clean:
- SQL: `$wpdb->prepare()` used correctly throughout
- CSRF: All handlers verify nonce
- Access Control: `current_user_can()` before every action
- etc.

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
```
