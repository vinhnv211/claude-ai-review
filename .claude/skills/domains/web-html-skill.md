---
name: web-html-skill
description: >
  Skill chuyên biệt cho review HTML templates trong embedded webgui,
  bao gồm form elements, input validation attributes, CSRF tokens,
  hidden fields, template escaping. Dùng bởi web-review-agent khi scope có file .html/.htm.
version: 0.1.0
---

# SKILL: Web HTML

## 1. PURPOSE
Tri thức và checklist cho review file HTML của webgui thiết bị nhúng.

## 2. WHEN TO USE
✅ Khi review file `.html`, `.htm`, template HTML (bao gồm output từ CGI generate HTML)

## 3. COMMON BUG PATTERNS

### Pattern 1: Stored XSS qua config value
```html
<!-- BUG: giá trị config render trực tiếp không escape -->
<input value="<?= uci.get("wireless", "@wifi-iface[0]", "ssid") ?>">
```
Nếu SSID chứa `"><script>alert(1)</script>` → XSS.
**Fix:** `luci.util.pcdata()` hoặc htmlspecialchars()

### Pattern 2: Form thiếu CSRF token
```html
<form action="/cgi-bin/apply.cgi" method="POST">
  <input name="password" type="password">
  <!-- Thiếu: <input type="hidden" name="csrf_token" value="..."> -->
</form>
```

### Pattern 3: Hidden field expose sensitive data
```html
<!-- BUG: expose admin flag trong client HTML -->
<input type="hidden" name="is_admin" value="1">
```
User có thể inspect DOM và manipulate.

### Pattern 4: Thiếu input constraints
```html
<!-- BUG: không giới hạn độ dài SSID (tối đa 32 bytes theo 802.11) -->
<input name="ssid" type="text">
<!-- Đúng: -->
<input name="ssid" type="text" maxlength="32" pattern="[^\x00-\x1f]+" required>
```

### Pattern 5: Form action sai method
```html
<!-- BUG: write operation dùng GET -->
<form action="/cgi-bin/reboot.cgi" method="GET">
```
GET request có thể bị trigger qua image src → CSRF.

## 4. REVIEW HEURISTICS
- Khi thấy `<?= ... ?>` hoặc `<%= %>` hoặc `{{...}}` → check escaping
- Khi thấy `<form>` → check CSRF token + method
- Khi thấy `<input type="hidden">` → check có rò rỉ thông tin nhạy cảm không
- Khi thấy input field không có `maxlength/pattern/min/max` → flag validation gap

## 5. HTML CHECKLIST
- [ ] Form action/method đúng (POST cho write)
- [ ] CSRF token present trong mọi form write
- [ ] Hidden fields không expose sensitive data
- [ ] Input fields có `maxlength`, `pattern`, `required` nếu cần
- [ ] Output escaping cho mọi dynamic content
- [ ] `<script>` tag không chứa user-controlled data chưa escape
- [ ] Không có inline event handlers nếu có CSP

## 6. SEVERITY GUIDELINE
- **critical**: XSS có thể exploit từ config input
- **major**: Missing CSRF token, GET dùng cho write
- **minor**: Thiếu maxlength, hidden field rò rỉ flag
- **info**: Inline style/script (CSP concerns)
