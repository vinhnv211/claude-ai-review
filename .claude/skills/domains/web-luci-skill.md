---
name: web-luci-skill
description: >
  Skill chuyên biệt cho review LuCI (OpenWrt web framework) — Lua controllers,
  templates, UCI binding, form validation, ACL entry. Dùng bởi web-review-agent
  khi scope có file .lua trong LuCI, hoặc HTM template trong `luasrc/view/`.
version: 0.1.0
---

# SKILL: Web LuCI / Lua (OpenWrt)

## 1. PURPOSE
Tri thức và checklist cho review LuCI pages trong OpenWrt:
- Controllers (`luasrc/controller/`)
- Models (`luasrc/model/cbi/`)
- Views / templates (`luasrc/view/`)
- UCI binding

## 2. WHEN TO USE
✅ Khi review file `.lua` trong `luasrc/`, `luci/`, hoặc template `.htm` của LuCI
✅ Khi review UCI set/get/commit qua Lua

## 3. COMMON BUG PATTERNS

### Pattern 1: `formvalue()` không validate
```lua
-- BUG: dùng thẳng vào UCI
local ssid = luci.http.formvalue("ssid")
uci:set("wireless", "@wifi-iface[0]", "ssid", ssid) -- không check nil, length, charset
```
**Fix:**
```lua
local ssid = luci.http.formvalue("ssid") or ""
if #ssid == 0 or #ssid > 32 or ssid:match("[\x00-\x1f]") then
  return error_response("Invalid SSID")
end
```

### Pattern 2: UCI set không commit
```lua
-- BUG: set xong không commit → thay đổi không persist
uci:set("network", "wan", "ipaddr", newip)
-- Thiếu: uci:commit("network")
```

### Pattern 3: UCI commit không kèm service reload
```lua
uci:set("wireless", ...)
uci:commit("wireless")
-- Thiếu: luci.sys.call("wifi reload")
```

### Pattern 4: Template không escape
```html
<!-- BUG trong .htm template -->
<input value="<%= luci.http.formvalue("ssid") %>">
```
**Fix:**
```html
<input value="<%= pcdata(luci.http.formvalue("ssid")) %>">
```
Hoặc dùng `%h` format trong `luci.util.string_format`.

### Pattern 5: ACL entry sai
```lua
-- BUG: entry không set ACL → mọi user (kể cả guest) truy cập được
function index()
  entry({"admin", "network", "wifi_advanced"}, call("action_save"))
  -- Thiếu: .dependent = true, ACL check
end
```

### Pattern 6: CSRF token bỏ qua
```lua
-- BUG: action không kiểm tra token
function action_apply()
  -- Thiếu: luci.http.formvalue("token") == luci.dispatcher.context.authtoken
  do_config_apply()
end
```
Trong LuCI, mọi POST write phải kiểm tra `token`.

### Pattern 7: Shell call với user input
```lua
-- BUG: command injection qua Lua
local iface = luci.http.formvalue("iface")
luci.sys.call("ifup " .. iface) -- iface="wan; reboot"
```
**Fix:**
```lua
if not iface:match("^[a-z0-9_]+$") then return error() end
luci.sys.call("ifup " .. iface)
```

### Pattern 8: Sai convention cho CBI model
```lua
-- BUG: CBI option thiếu datatype → input không validate
s:option(Value, "ip", "IP")
-- Đúng:
ip = s:option(Value, "ip", "IP")
ip.datatype = "ip4addr"
```

## 4. REVIEW HEURISTICS
- `luci.http.formvalue(`, `luci.http.getenv(` → trace validation
- `uci:set(` không kèm `uci:commit(` → config không persist
- `uci:commit(` không kèm service reload → state inconsistent
- `luci.sys.call(` / `os.execute(` với concat string → command injection
- Template `<%= ... %>` không có `pcdata()` → XSS
- CBI `Value` không có `datatype` → missing validation
- Controller `entry()` không có ACL check → unauthorized access

## 5. LUCI CHECKLIST
- [ ] Mọi `formvalue()` đều validate trước khi dùng (nil, length, charset)
- [ ] UCI `set` → `commit` → service reload theo đúng chain
- [ ] Template escape bằng `pcdata()` cho dynamic content
- [ ] CBI options có `datatype` phù hợp (`ip4addr`, `uinteger`, etc.)
- [ ] Controller `entry()` có ACL / dependent check
- [ ] Write actions check CSRF token
- [ ] Không dùng `luci.sys.call()` với user input chưa validate
- [ ] Error handling cho UCI write fail (disk full, readonly fs)
- [ ] Không expose sensitive config (password, key) trong view

## 6. SEVERITY GUIDELINE
- **critical**: `luci.sys.call(user_input)`, template XSS exploit được
- **major**: Missing CSRF token, missing ACL, UCI không commit
- **minor**: CBI thiếu datatype, template chưa dùng pcdata nhưng data trusted
- **info**: Code style, unused variables
