# DevVN WordPress Skills

A collection of [Agent Skills](https://agentskills.io) for WordPress plugin and theme development. Built by [Lê Văn Toản](https://levantoan.com) — battle-tested across 20+ commercial WordPress plugins.

## Skills

| Skill | Description |
|-------|-------------|
| [`devvn-wp-security-audit`](skills/devvn-wp-security-audit/) | Comprehensive WPCS security audit — SQL Injection, XSS, CSRF, Access Control, File Upload. Scans all PHP files with file:line reporting and fix suggestions. |

## Installation

### All skills (recommended)

Clone the repo, then symlink or copy the `skills/` contents into your Claude skills directory:

```bash
git clone https://github.com/levantoan/DevVN-WordPress-Skills ~/DevVN-WordPress-Skills

# Linux / macOS — symlink each skill
for skill in ~/DevVN-WordPress-Skills/skills/*/; do
  ln -s "$skill" ~/.claude/skills/$(basename "$skill")
done

# Windows (PowerShell) — symlink each skill
Get-ChildItem "$HOME\DevVN-WordPress-Skills\skills" -Directory | ForEach-Object {
  New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\$($_.Name)" -Target $_.FullName
}
```

### Single skill

```bash
git clone --depth 1 https://github.com/levantoan/DevVN-WordPress-Skills /tmp/devvn-skills
cp -r /tmp/devvn-skills/skills/devvn-wp-security-audit ~/.claude/skills/
rm -rf /tmp/devvn-skills
```

### Project-level (shared with your team via git)

```bash
cp -r ~/DevVN-WordPress-Skills/skills/devvn-wp-security-audit .claude/skills/
git add .claude/skills/
git commit -m "Add DevVN security audit skill"
```

### Update

```bash
cd ~/DevVN-WordPress-Skills && git pull
```

Skills are symlinked, so `git pull` updates everything automatically.

## Usage

Skills activate automatically when Claude detects a matching task, or invoke directly:

```
/devvn-wp-security-audit        # Run full security audit on current project
```

## Compatibility

Works with any tool that supports the [Agent Skills](https://agentskills.io) open standard:

**Claude Code** · **Cursor** · **VS Code Copilot** · **GitHub Copilot** · **Gemini CLI** · **OpenCode** · **Goose** · **Kiro** · **Roo Code** · and [many more](https://agentskills.io/home)

## Directory Structure

```
DevVN-WordPress-Skills/
├── README.md
└── skills/
    └── devvn-wp-security-audit/
        ├── SKILL.md
        └── references/
            ├── audit-rules.md
            └── report-format.md
```

## Contributing

1. Fork this repository
2. Create your skill directory inside `skills/` with a `SKILL.md` file (follow the [Agent Skills spec](https://agentskills.io/specification))
3. Add detailed documentation in `references/`
4. Submit a pull request

## License

MIT

---

# DevVN WordPress Skills (Tiếng Việt)

Bộ sưu tập [Agent Skills](https://agentskills.io) dành cho phát triển plugin và theme WordPress. Được xây dựng bởi [Lê Văn Toản](https://levantoan.com) — đã kiểm chứng qua 20+ plugin WordPress thương mại.

## Danh sách Skills

| Skill | Mô tả |
|-------|-------|
| [`devvn-wp-security-audit`](skills/devvn-wp-security-audit/) | Audit bảo mật toàn diện theo chuẩn WPCS — SQL Injection, XSS, CSRF, Access Control, File Upload. Quét tất cả file PHP, báo cáo chi tiết file:line kèm gợi ý sửa lỗi. |

## Cài đặt

### Tất cả skills (khuyên dùng)

Clone repo, sau đó symlink hoặc copy nội dung thư mục `skills/` vào thư mục skills của Claude:

```bash
git clone https://github.com/levantoan/DevVN-WordPress-Skills ~/DevVN-WordPress-Skills

# Linux / macOS — symlink từng skill
for skill in ~/DevVN-WordPress-Skills/skills/*/; do
  ln -s "$skill" ~/.claude/skills/$(basename "$skill")
done

# Windows (PowerShell) — symlink từng skill
Get-ChildItem "$HOME\DevVN-WordPress-Skills\skills" -Directory | ForEach-Object {
  New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\$($_.Name)" -Target $_.FullName
}
```

### Một skill duy nhất

```bash
git clone --depth 1 https://github.com/levantoan/DevVN-WordPress-Skills /tmp/devvn-skills
cp -r /tmp/devvn-skills/skills/devvn-wp-security-audit ~/.claude/skills/
rm -rf /tmp/devvn-skills
```

### Cấp project (chia sẻ với team qua git)

```bash
cp -r ~/DevVN-WordPress-Skills/skills/devvn-wp-security-audit .claude/skills/
git add .claude/skills/
git commit -m "Add DevVN security audit skill"
```

### Cập nhật

```bash
cd ~/DevVN-WordPress-Skills && git pull
```

Nếu dùng symlink thì `git pull` tự cập nhật tất cả skills.

## Cách sử dụng

Skills tự động kích hoạt khi Claude phát hiện task phù hợp, hoặc gọi trực tiếp:

```
/devvn-wp-security-audit        # Chạy audit bảo mật toàn bộ project
```

## Tương thích

Hoạt động với mọi công cụ hỗ trợ chuẩn [Agent Skills](https://agentskills.io):

**Claude Code** · **Cursor** · **VS Code Copilot** · **GitHub Copilot** · **Gemini CLI** · **OpenCode** · **Goose** · **Kiro** · **Roo Code** · và [nhiều hơn nữa](https://agentskills.io/home)

## Đóng góp

1. Fork repository này
2. Tạo thư mục skill bên trong `skills/` với file `SKILL.md` (theo [Agent Skills spec](https://agentskills.io/specification))
3. Thêm tài liệu chi tiết trong `references/`
4. Gửi pull request

## Giấy phép

MIT
