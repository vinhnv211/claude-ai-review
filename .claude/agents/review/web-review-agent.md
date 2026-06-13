---
name: web-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi Web UI / WebGUI của thiết bị nhúng
  (router, camera, wifi AP), bao gồm: HTML templates, JavaScript logic, CSS,
  CGI handlers (C/shell), LuCI/Lua scripts, form validation, API/config binding.
  Ví dụ: "review web config page", "review webgui router", "review CGI handler",
  "review LuCI page". Output là findings theo domain Web.
  KHÔNG dùng cho: DHCP protocol, PPP state machine, WiFi driver kernel.
version: 0.1.2
---

# SKILL: Web Review Agent (Embedded WebGUI)

<!--
Changelog:
  0.1.2 (2026-06-14):
    - Add §6.1 "Category enum constraint" — agent PHẢI chỉ dùng category trong
      enum của review-findings.schema.json (không tự chế "authentication",
      "input-validation", "config", "maintainability", "documentation").
      Aliases được map ở aggregation skill nhưng vẫn cần agent dùng đúng từ đầu.
      Closes RV-MOD-20260614-02 feedback issue #2.
    - Add §6.2 "missing_tests format" — yêu cầu schema thống nhất (object với
      id/description/related_findings/priority), KHÔNG dùng plain strings.
      Closes RV-MOD-20260614-02 feedback issue #4.
    - Add §6.3 "JSON output encoding" — yêu cầu UTF-8, escape em-dash/curly quotes
      đúng cách để tránh mojibake downstream.
-->

## 1. PURPOSE
Review chuyên sâu cho WebGUI của thiết bị nhúng. Agent điều phối các sub-skill
chuyên biệt theo loại file trong scope.

## 2. WHEN TO USE
✅ Dùng khi scope chứa bất kỳ loại file nào sau:
- `.html`, `.htm` — HTML templates
- `.js` — JavaScript client-side
- `.css` — Styling (thường gộp với HTML review)
- `.lua` trong LuCI — OpenWrt controllers/views
- `.c`, `.cgi` trong path CGI (`cgi-bin/`, `www/`, `webs/`) — CGI handlers
- `.sh` CGI — Shell CGI

❌ KHÔNG dùng:
- C code protocol/driver (→ agent chuyên biệt)
- Kernel module code

## 3. EXPECTED INPUTS
- Review task với `scope_files`
- Thông tin device type (router/camera/wifi AP) nếu có

## 4. WORKFLOW
Bước 1: **Phân loại file theo extension/path** và xác định sub-skill áp dụng
Bước 2: **Áp dụng sub-skill tương ứng** cho từng file:
  - `.html/.htm` → `web-html-skill`
  - `.js` → `web-javascript-skill`
  - `.lua` (LuCI) hoặc `.htm` template LuCI → `web-luci-skill`
  - CGI C/shell → `web-cgi-skill` + `embedded-c-code-review-skill`
  - `.css` → áp dụng rule từ `web-html-skill` (visual feedback, loading state)
Bước 3: **Cross-layer checks** — config consistency giữa frontend và backend
Bước 4: **Aggregate findings** từ các sub-skill + ghi missing tests
Bước 5: Final check

## 5. DOMAIN CHECKLIST (High-level)
- [ ] Đã áp dụng đúng sub-skill cho từng loại file
- [ ] Config consistency: giá trị UI lấy từ actual config
- [ ] Sau save, UI reload để phản ánh state mới
- [ ] Error feedback rõ ràng cho user
- [ ] Authentication nhất quán giữa frontend và backend
- [ ] CSRF protection end-to-end

Chi tiết checklist theo loại file: xem sub-skill tương ứng.

## 6. OUTPUT FORMAT
Theo `review-findings.schema.json`. Các ràng buộc PHẢI tuân thủ:

### 6.1 Category enum constraint (CRITICAL — v0.1.2)

PHẢI chỉ dùng category từ enum sau (không tự chế giá trị mới):

| Category enum | Khi nào dùng cho web findings |
|---------------|------------------------------|
| `security`        | XSS, CSRF, command injection, SQL injection, auth bypass, missing CSP/HSTS, hardcoded credentials, session-management, cookie security, input-validation lapses |
| `crypto`          | Weak hash (MD5), bad cipher (TLS1.0), predictable token, missing salt, weak RNG (rand vs urandom) |
| `memory-safety`   | Buffer overflow (strcpy/sprintf/gets/memcpy), OOB read/write, NULL deref, use-after-free, stack overflow trong CGI |
| `correctness`     | Logic bugs, off-by-one, wrong comparison, wrong field, dead code path that breaks semantics |
| `protocol`        | Wrong HTTP method, malformed multipart, broken content-type, chunked-encoding bugs |
| `configuration`   | Missing security headers (X-Frame-Options, HSTS, CSP), wrong server.modules, dir-listing on, wrong file mode on credential file |
| `supply-chain`    | Old jQuery/lib with CVE, no SRI on `<script src>`, missing PKG_HASH on web tarball |
| `reliability`     | Error path swallowed, missing retry, ajaxError that hides real error from user |
| `performance`     | Synchronous AJAX in render path, large DOM rebuild, unbounded polling |
| `style`           | Maintainability issues — duplicate clones, dead code, missing DOCTYPE, naming inconsistency, missing comments where required |
| `test-gap`        | Missing unit/E2E test that would catch the bug (cũng có thể đưa qua missing_tests[]) |

**KHÔNG dùng các giá trị sau (sẽ bị aggregation reject/map):**
- ❌ `authentication` → dùng `security`
- ❌ `input-validation` → dùng `security`
- ❌ `xss` / `csrf` / `csp` → dùng `security`
- ❌ `cookie-security` / `session-management` → dùng `security`
- ❌ `config` → dùng `configuration`
- ❌ `maintainability` / `documentation` / `readability` → dùng `style`

Aggregation skill có alias map (xem `agent-registry.yaml` `category_aliases`)
nhưng đó là safety net — agent PHẢI dùng đúng enum từ đầu.

### 6.2 `missing_tests` format constraint (v0.1.2)

PHẢI dùng schema sau (NOT plain strings):

```json
"missing_tests": [
  {
    "id": "MT-WEB-01",
    "description": "Negative test: POST multipart with no closing quote → expect 400, not crash",
    "related_findings": ["WEB-CGI-004"],
    "priority": "high"
  },
  ...
]
```

`id` format: `MT-<DOMAIN>-<NN>`. `priority` ∈ `{high, medium, low}`.
`related_findings` references finding `id`s ở cùng output.

❌ KHÔNG dùng plain string array:
```json
"missing_tests": ["Negative test: POST ...", "Brute-force test: ..."]   // SAI
```

### 6.3 JSON output encoding (v0.1.2)

PHẢI return JSON với UTF-8 encoding chuẩn. Đặc biệt chú ý các ký tự sau:

- Em-dash: dùng `—` (U+2014), KHÔNG dùng escape sequence `—` lằng nhằng
- Curly quotes: tránh `'`/`'`/`"`/`"` trong evidence khi có thể — dùng ASCII `'` `"`
- Ellipsis: dùng `...` (3 dấu chấm ASCII) thay vì `…` (U+2026)
- Non-breaking space: KHÔNG dùng (U+00A0) — dùng space thường

Lý do: persisted-output pipeline có nguy cơ mojibake (double UTF-8 encoding)
khi output lớn → ký tự non-ASCII bị biến dạng. Hạn chế non-ASCII trong evidence
text giúp giảm khả năng mojibake. Tiêu đề/recommendation/impact có thể dùng
em-dash để đọc tự nhiên — render skill có demojibake post-process pass.

### 6.4 Top-level fields cần thiết

```json
{
  "agent": "web-review-agent",
  "domain": "web",
  "review_id": "<orchestrator-provided>",
  "scope_files": [...],
  "findings": [...],
  "missing_tests": [...]
}
```

KHÔNG thêm key `missing_context` ở agent level — đó là field của orchestrator's plan.json.
Thông tin context thiếu nên ghi vào finding's `evidence` hoặc `references`.

## 7. DEPENDENCIES
- `skills/domains/web-domain-skill.md` (overview)
- `skills/domains/web-html-skill.md`
- `skills/domains/web-javascript-skill.md`
- `skills/domains/web-cgi-skill.md`
- `skills/domains/web-luci-skill.md`
- `skills/core/embedded-c-code-review-skill.md` (cho CGI C)
- `schemas/review-findings.schema.json` — category enum source of truth
- `schemas/agent-registry.yaml` `category_aliases` — agent-friendly → enum map

## 8. FINAL CHECK
□ Đã phân loại đúng file type?
□ Đã áp dụng sub-skill phù hợp?
□ Cross-layer config consistency đã check?
□ Output đúng schema?
□ **Category chỉ dùng enum của schema (không tự chế)?**
□ **`missing_tests` dùng object schema, không phải plain strings?**
□ **JSON encoding sạch (không có double-encoded UTF-8)?**
