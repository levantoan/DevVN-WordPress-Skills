# Field Mapping Guide

> All field types below work identically with ACF Pro, ACF Free, and SCF (Secure Custom Fields). Repeater fields require ACF Pro or SCF (included free in SCF).

## How to analyze an HTML page and map content to ACF/SCF field types

### Step 1: Identify sections

Scan the HTML for semantic sections. Common patterns:

- `<section>` or `<div>` with id/class like `hero`, `features`, `products`, `services`, `contact`, `faq`, `footer`
- Each section typically maps to one **ACF tab**

### Step 2: Classify each content element

| HTML Pattern | ACF Field Type | Notes |
|---|---|---|
| Single heading text | `text` | Use for h1, h2, section titles |
| Short paragraph | `text` | One-liner descriptions, badge text |
| Long paragraph | `textarea` | Multi-line descriptions |
| Image `<img src="...">` | `image` (return_format: url) | Product photos, icons |
| Background image in CSS | `image` (return_format: url) | Hero bg, stats bg — output via inline `style=""` |
| `<form>` element | `textarea` | User pastes shortcode from form plugin (CF7, WPForms...). Render with `do_shortcode()`. Never hardcode forms |
| Phone number | `text` | Store formatted, strip to digits with `preg_replace` for `tel:` links |
| Email | `text` (or `email`) | Used in both display and `mailto:` |
| URL / link | `url` | Social media links, external links |
| Color choice (limited options) | `select` or `radio` | Badge colors, style variants |
| Rich content with links | `wysiwyg` (toolbar: basic) | Footer menu columns, legal links |
| Repeating cards/items | `repeater` | Features grid, product cards, FAQ, stats |

### Step 3: Map repeater sub-fields

For each repeated element, identify sub-fields:

**Feature card:**
```
repeater: features
  - icon (image)
  - title (text)
  - desc (textarea)
```

**Product card:**
```
repeater: products
  - image (image)
  - badge_text (text)
  - badge_color (select: purple/green/amber)
  - title (text)
  - desc (textarea)
  - features (textarea — one per line)
  - link (text)
```

**FAQ item:**
```
repeater: faq
  - question (text)
  - answer (textarea)
```

**Stats counter:**
```
repeater: stats
  - value (number)
  - suffix (text — e.g., "+")
  - label (text)
```

**Gallery/project item:**
```
repeater: projects
  - image (image)
  - name (text)
```

**Footer menu column:**
```
repeater: footer_menus
  - title (text)
  - content (wysiwyg, toolbar: basic)
```

### Step 4: Organize into tabs

Group related fields into ACF tabs. Each tab corresponds to a visual section of the page:

```
Tab: Cơ bản       → logo
Tab: Hero          → bg image, badge, titles, CTA, phone, form text, stats repeater
Tab: Đặc điểm     → badge, title, desc, features repeater
Tab: Sản phẩm     → badge, title, desc, products repeater
Tab: Dịch vụ      → badge, title, desc, services repeater
Tab: Tại sao chọn → badge, title, desc, reasons repeater (with zigzag images)
Tab: Thống kê     → bg image, stats repeater
Tab: Dự án        → badge, title, desc, projects repeater
Tab: Bảng màu     → badge, title, desc, colors repeater, CTA
Tab: Liên hệ      → badge, title, desc, phone, email, address, form title
Tab: FAQ           → badge, title, FAQ repeater
Tab: Footer        → desc, social URLs, menu columns repeater, copyright, legal
```

### Step 5: Naming conventions

Use a consistent prefix for all fields to avoid conflicts:

- Field keys: `field_{prefix}_{section}_{name}` — e.g., `field_ldp_hero_title`
- Field names: `{prefix}_{name}` — e.g., `ldp_hero_title`
- Sub-field keys: `field_{prefix}_{section_abbr}_{name}` — e.g., `field_ldp_fi_icon`
- Sub-field names: short descriptive — e.g., `icon`, `title`, `desc`

The prefix should be unique per template to avoid collisions if multiple landing pages exist.
