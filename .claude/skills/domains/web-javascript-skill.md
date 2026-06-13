---
name: web-javascript-skill
description: >
  Skill chuyên biệt cho review JavaScript trong embedded webgui,
  bao gồm AJAX/fetch calls, DOM manipulation, client-side validation,
  XSS qua innerHTML/eval, error handling, double-submit protection.
  Dùng bởi web-review-agent khi scope có file .js.
version: 0.1.0
---

# SKILL: Web JavaScript

## 1. PURPOSE
Tri thức và checklist cho review file JavaScript của webgui thiết bị nhúng.
Các webgui router/camera thường dùng vanilla JS hoặc jQuery (không phải framework lớn).

## 2. WHEN TO USE
✅ Khi review file `.js` hoặc inline `<script>` trong HTML

## 3. COMMON BUG PATTERNS

### Pattern 1: DOM XSS qua innerHTML
```javascript
// BUG: data từ API render trực tiếp
fetch('/api/status').then(r => r.json()).then(d => {
  document.getElementById('msg').innerHTML = d.message; // XSS nếu message chứa HTML
});
```
**Fix:** dùng `textContent` thay vì `innerHTML`, hoặc escape HTML entities.

### Pattern 2: eval() với user data
```javascript
// BUG: eval config value
let config = eval("(" + localStorage.getItem("cfg") + ")");
```
**Fix:** dùng `JSON.parse()`.

### Pattern 3: Missing AJAX error handling
```javascript
// BUG: không handle lỗi
$.post("/api/apply", data, function(resp) {
  alert("Saved");
}); // Nếu timeout/500/403 → user không biết
```
**Fix:**
```javascript
$.post(...).done(...).fail(function(xhr) {
  showError("Lỗi: " + xhr.statusText);
});
```

### Pattern 4: Double-submit (race condition)
```javascript
// BUG: button không disable sau click
$("#save").click(function() {
  $.post("/api/save", formData);
});
```
**Fix:** disable button + show spinner, re-enable sau khi response về.

### Pattern 5: Missing client-side validation trước submit
```javascript
// BUG: gửi bất kỳ giá trị gì
function save() {
  $.post("/api/set", {ip: $("#ip").val()});
}
```
**Fix:** validate IP format, length, range trước khi gọi API.

### Pattern 6: No timeout handling
```javascript
// BUG: AJAX không có timeout → user treo nếu backend hang
$.ajax({url: '/api/...', success: ...});
```
**Fix:** `$.ajax({timeout: 10000, ...})` + xử lý timeout case.

### Pattern 7: Hardcoded credentials hoặc debug flags
```javascript
// BUG
const API_KEY = "abc123"; // expose trong JS bundle
const DEBUG = true;       // bật debug UI trong production
```

### Pattern 8: Race trong multi-step workflow
```javascript
// BUG: không đảm bảo bước trước hoàn tất
saveWifi();
saveNetwork();   // có thể bắt đầu trước khi saveWifi xong
reboot();        // có thể trigger trước khi save xong
```
**Fix:** dùng Promise chain hoặc async/await.

## 4. REVIEW HEURISTICS
- `innerHTML =`, `document.write`, `insertAdjacentHTML` → check XSS
- `eval(`, `new Function(` → flag critical
- `.post(`, `.ajax(`, `fetch(` → check `.fail` / `.catch` + timeout
- Button click handler không disable → double-submit risk
- Hardcoded constants in caps → có phải credential không?

## 5. JS CHECKLIST
- [ ] Không dùng `eval()` / `new Function()` với user data
- [ ] Không `innerHTML` với data không escape
- [ ] AJAX có `.fail()` / `.catch()` handler
- [ ] AJAX có timeout + xử lý timeout case
- [ ] Form submit disable button prevent double-submit
- [ ] Client-side validation trước submit
- [ ] Không hardcoded credentials / debug flags
- [ ] Multi-step async workflow dùng Promise chain
- [ ] Loading indicator cho long operations

## 6. SEVERITY GUIDELINE
- **critical**: `eval(user_data)`, DOM XSS exploit được
- **major**: Missing error handling, double-submit, race condition
- **minor**: Missing timeout, missing loading indicator
- **info**: Hardcoded debug flag, dead code
