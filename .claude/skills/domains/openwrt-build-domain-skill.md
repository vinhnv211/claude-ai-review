---
name: openwrt-build-domain-skill
description: >
  Tri thức domain cho OpenWrt package build: Makefile patterns, patches series,
  overlay files (init, uci-defaults, hotplug). Dùng bởi openwrt-build-review-agent.
version: 0.1.0
---

# SKILL: OpenWrt Build Domain

## 1. PURPOSE
Cung cấp bug patterns, heuristics và checklist cho review build-level của OpenWrt
package. Không cover source code upstream — phần đó thuộc domain agent tương ứng.

## 2. ARCHITECTURE TÓM TẮT

```
OpenWrt buildroot
  └─ package/<category>/<pkg>/
       ├─ Makefile            ← fetch + build + install rule
       ├─ patches/            ← áp lên tarball extract
       │   ├─ 001-xxx.patch
       │   └─ 002-yyy.patch
       └─ files/              ← copy vào image
           ├─ etc/config/<pkg>
           ├─ etc/init.d/<pkg>
           ├─ etc/uci-defaults/<pkg>
           └─ etc/hotplug.d/**/<pkg>

Build flow:
  fetch tarball → verify hash → extract → apply patches →
  configure → compile → install tới staging → package vào .ipk
```

## 3. MAKEFILE BUG PATTERNS

### 3.1 Hash mismatch (CRITICAL)
```makefile
PKG_VERSION:=1.2.4
PKG_HASH:=abc...  ← của version 1.2.3
```
**Impact:** Build fail hoặc tệ hơn: nếu có MITM URL, attacker có thể serve tarball khác.
**Check:** Hash phải match tarball tương ứng PKG_VERSION.

### 3.2 Thiếu PKG_HASH (CRITICAL security)
```makefile
PKG_SOURCE:=apps-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://foo.com/
# PKG_HASH chưa có
```
**Impact:** Supply chain attack — tarball bị modify không detect được.

### 3.3 MD5 legacy thay vì SHA256 (MAJOR)
```makefile
PKG_MD5SUM:=deadbeef...
```
**Impact:** MD5 đã broken. Phải dùng `PKG_HASH` (SHA256).

### 3.4 Thiếu DEPENDS (MAJOR)
```makefile
define Package/apps
  DEPENDS:=+libc   ← thiếu +libssl dù source dùng OpenSSL
endef
```
**Impact:** Build trên clean environment fail, hoặc runtime link error.
**Check:** Cross-reference `#include <...>` trong source với DEPENDS.

### 3.5 Hardcode host path (MAJOR)
```makefile
CONFIGURE_ARGS+=--with-ssl=/usr/local/openssl
```
**Impact:** Build reproducibility fail, leak path của maintainer.
**Fix:** Dùng `$(STAGING_DIR)/usr/lib/...`.

### 3.6 Race trong install (MAJOR)
```makefile
define Package/apps/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) ... $(1)/usr/bin/
	chmod 4755 $(1)/usr/bin/apps   ← setuid!
endef
```
**Impact:** Setuid binary — security risk nếu không cần.

### 3.7 postinst chạy không idempotent (MINOR)
```bash
#!/bin/sh
uci set apps.@config[0].enabled=1   ← tạo mới mỗi lần upgrade
uci commit apps
```
**Fix:** Check section tồn tại trước, dùng uci-defaults pattern.

## 4. PATCH QUALITY BUG PATTERNS

### 4.1 Patch fuzzy apply (MAJOR)
`patch -p1` chạy với warning "fuzz 2" → patch rebase kém, dễ gãy lần sau.
**Fix:** Refresh patch với `make package/<pkg>/refresh`.

### 4.2 Patch sửa file generated (MAJOR)
Patch sửa `configure`, `Makefile.in`, `ltmain.sh` → sẽ mất khi autoreconf.
**Fix:** Patch source (`configure.ac`, `Makefile.am`) thay vì generated.

### 4.3 Missing Signed-off-by (MINOR)
Patch không có header upstream-ready → khó submit lên upstream.
**Fix:** Thêm `From: / Subject: / Signed-off-by:` dùng `git format-patch`.

### 4.4 Overlap giữa các patch (MAJOR)
Patch 002 modify lại dòng patch 001 đã thêm → fragile, khó maintain.
**Fix:** Squash hoặc reorder.

### 4.5 Patch không reference upstream (MINOR)
Không biết patch này đã upstream chưa → stale patch tích lũy.
**Fix:** Add comment `Upstream: <commit-hash>` hoặc `Upstream-Status: Pending`.

### 4.6 Patch cho CVE không có reference (MAJOR)
Patch security mà không ghi CVE → review không biết tầm quan trọng.
**Fix:** `Subject: CVE-2024-xxxx: description`.

## 5. INIT SCRIPT BUG PATTERNS

### 5.1 Priority trùng system service (MAJOR)
```sh
START=10   ← system service range
```
**Fix:** Dùng 50-99 cho user service.

### 5.2 Không reload config (MINOR)
```sh
service_triggers() {
    # thiếu
}
```
→ thay đổi UCI không tự reload service.
**Fix:**
```sh
service_triggers() {
    procd_add_reload_trigger "apps"
}
```

### 5.3 Block trong start (MAJOR)
```sh
start_service() {
    while ! ping -c1 8.8.8.8; do sleep 1; done   ← block boot
    ...
}
```
**Fix:** Dùng hotplug hoặc procd instance deps.

### 5.4 Hardcode binary path (MINOR)
```sh
/usr/bin/apps --config /etc/apps.conf
```
**Fix:** Dùng biến `$PROG`, hoặc `procd_set_param command $PROG`.

## 6. UCI-DEFAULTS BUG PATTERNS

### 6.1 Không exit 0 (MAJOR)
Script fail → OpenWrt retry mỗi boot.
**Fix:**
```sh
#!/bin/sh
uci -q batch <<EOF
...
EOF
exit 0
```

### 6.2 Rerun không idempotent (MAJOR)
Mỗi lần chạy tạo thêm section mới.
**Fix:** Check trước:
```sh
[ -n "$(uci -q get apps.@config[0])" ] || uci add apps config
```

### 6.3 Gọi service chưa chạy (MINOR)
```sh
/etc/init.d/apps reload   ← service chưa enabled
```
**Fix:** Chỉ gọi sau khi enable.

## 7. HOTPLUG BUG PATTERNS

### 7.1 Không check $ACTION (MAJOR)
```sh
#!/bin/sh
# xử lý cả ifup lẫn ifdown như nhau
do_something
```
**Fix:**
```sh
[ "$ACTION" = "ifup" ] || exit 0
```

### 7.2 Block dài (MAJOR)
Hotplug script chạy sync, block sẽ delay cả network bring-up.
**Fix:** Nếu cần xử lý dài → dispatch vào background hoặc dùng procd.

## 8. SEVERITY GUIDELINE

| Bug | Severity |
|-----|----------|
| Thiếu PKG_HASH / hash mismatch | critical |
| MD5 thay vì SHA256 | major |
| DEPENDS thiếu library thực dùng | major |
| Patch fuzzy / sửa generated file | major |
| Setuid không justify | major |
| uci-defaults không exit 0 | major |
| init priority trùng system | major |
| Patch thiếu Signed-off-by | minor |
| Hardcode binary path trong init | minor |
| Missing PKG_LICENSE | minor |
| Style (tên patch không NNN-) | minor |

## 9. TEST GAPS TO FLAG

- Package build được trên clean buildroot không?
- Package install vào image không lỗi?
- Upgrade từ version cũ preserve config không?
- uci-defaults chạy thành công trên first boot?
- init script start/stop/restart/reload đều work?
- Patches apply sạch khi bump PKG_VERSION?

## 10. DEPENDENCIES
- Hiểu cơ bản OpenWrt buildroot, procd, uci
- Quen với quilt (OpenWrt dùng quilt để maintain patch series)
