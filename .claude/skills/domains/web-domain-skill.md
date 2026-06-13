---
name: web-domain-skill
description: >
  Skill overview cho WebGUI của thiết bị nhúng (router/camera/wifi AP).
  Cung cấp kiến trúc tổng thể, file types, và định hướng tới sub-skill chuyên biệt
  (web-html, web-javascript, web-cgi, web-luci).
version: 0.1.0
---

# SKILL: Web Domain (Overview)

## 1. PURPOSE
Overview knowledge về WebGUI của thiết bị nhúng. Skill này là **điểm tham chiếu**
cho các sub-skill chuyên biệt.

Chi tiết bug patterns và checklist được đặt trong các sub-skill:
- `web-html-skill` — HTML templates, forms, CSRF, escaping
- `web-javascript-skill` — AJAX, DOM, XSS, validation, double-submit
- `web-cgi-skill` — CGI C/shell, command injection, buffer overflow, auth
- `web-luci-skill` — OpenWrt LuCI Lua, UCI binding, CBI, ACL

## 2. EMBEDDED WEBGUI ARCHITECTURE

```
Browser ←→ uhttpd/lighttpd ←→ CGI (C/shell)     ←→ UCI/config files
                            ←→ LuCI (Lua)        ←→ UCI/ubus
                            ←→ REST API (C)      ←→ backend daemon
```

## 3. FILE TYPES
| Extension | Vai trò | Sub-skill |
|-----------|---------|-----------|
| `.html/.htm` | Template tĩnh / CGI output | `web-html-skill` |
| `.js`        | Client logic, AJAX         | `web-javascript-skill` |
| `.css`       | Styling                    | `web-html-skill` (gộp) |
| `.lua`       | LuCI controller + template | `web-luci-skill` |
| `.c` CGI     | Handler biên dịch          | `web-cgi-skill` + `embedded-c-code-review-skill` |
| `.sh`        | Shell CGI (thường rủi ro)  | `web-cgi-skill` |
| `.cgi`       | Binary CGI                 | `web-cgi-skill` (nếu có source) |

## 4. DEVICE-SPECIFIC RISKS

### Router WebGUI
- WAN/LAN form: validate IP/mask/gateway format
- WiFi: SSID ≤ 32 bytes, password ≤ 63 bytes (WPA2)
- Firewall rules: input validation cho port, IP range
- Port forwarding: check duplicate, valid range

### Camera WebGUI
- RTSP stream URL: không inject vào shell
- Recording schedule: validate date/time format
- Snapshot path: không path traversal
- User/password management: weak password policy

### WiFi AP
- Channel selection: validate theo regulatory domain
- TX power: validate theo hardware limit
- SSID broadcast: state sync với driver state

## 5. TOP-LEVEL SECURITY RISKS (tổng quan)
| Risk | Severity | Sub-skill chi tiết |
|------|----------|--------------------|
| Command injection qua CGI | critical | web-cgi-skill |
| Auth bypass | critical | web-cgi-skill, web-luci-skill |
| Stored XSS | critical | web-html-skill, web-luci-skill |
| DOM XSS | critical | web-javascript-skill |
| CSRF | major | web-html-skill, web-luci-skill |
| Buffer overflow CGI | critical | web-cgi-skill |
| Missing validation | major | web-html-skill, web-javascript-skill |
| Double submit | minor | web-javascript-skill |

## 6. CROSS-LAYER CHECKS
Các check vượt ra ngoài phạm vi một sub-skill:
- Config read-back: UI hiển thị giá trị thực từ backend chưa?
- Save flow consistency: UI → API → UCI → commit → service reload — chain có đủ bước?
- Auth consistency: session check ở frontend và backend phải khớp
- CSRF token: frontend gửi và backend validate
- Error propagation: backend error → user-visible message

## 7. COMMON TEST GAPS
- Special characters trong SSID field
- Maximum field length edge cases
- Concurrent config saves từ nhiều tab
- Network timeout khi apply config
- Session expiry behavior
- Browser back button sau form submit
