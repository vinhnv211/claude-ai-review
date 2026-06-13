---
name: openwrt-build-review-agent
description: >
  Skill này nên được dùng khi cần review các file build-level của OpenWrt package:
  Makefile (PKG_SOURCE, PKG_HASH, define Package, dependencies), patches/*.patch
  (quality, upstream-ability, conflict risk), và files/** overlay (init scripts,
  hotplug, uci-defaults). Thường chạy song song với domain agent review source thực.
  KHÔNG dùng cho: source code trong tarball (để openwrt-source-context-skill trích
  xuất rồi route tới domain agent), kernel module review (wifi-driver-review-agent).
version: 0.1.0
---

# SKILL: OpenWrt Build Review Agent

## 1. PURPOSE
Review chất lượng build của một OpenWrt package: Makefile, patches series, overlay files.
Đây là các file **do OpenWrt packaging yêu cầu**, không phải source upstream — nên có
checklist và bug pattern riêng, tách khỏi review source code.

## 2. WHEN TO USE

✅ Dùng khi scope chứa:
- `package/**/Makefile` (OpenWrt package Makefile)
- `package/**/patches/*.patch`
- `package/**/files/**` (init.d, hotplug.d, uci-defaults, config/)
- `package/base-files/**` (bao gồm `/lib/functions.sh`, `/lib/upgrade/`, `/sbin/*`)
- `**/etc/init.d/*`, `**/etc/uci-defaults/*`, `**/etc/hotplug.d/**`
- `**/lib/functions.sh`, `**/lib/functions/*.sh`, `**/lib/upgrade/*.sh`
- `scripts/**/*.sh` (buildroot scripts)

❌ KHÔNG dùng:
- Source C/H/Lua trong tarball (đã extract) — để domain agent handle
- Top-level `Makefile` / `rules.mk` của OpenWrt buildroot
- Kernel driver source

## 3. EXPECTED INPUTS
- `review-task` với `scope_files` thuộc các path trên
- Ngữ cảnh từ `openwrt-source-context-skill` (khi cần): `pkg_name`, `pkg_version`,
  `patches_applied`, `patches_failed`

## 4. WORKFLOW

**Bước 1: Phân loại file**
- `Makefile` → áp `openwrt-build-domain-skill` §3 (Makefile patterns)
- `*.patch` → áp §4 (Patch quality)
- `files/*.init` / `files/etc/init.d/*` → áp §5 (init script)
- `files/etc/uci-defaults/*` → áp §6 (uci-defaults)
- `files/etc/hotplug.d/**` → áp §5 (hotplug)
- `files/**/*.sh`, base-files shell, init.d/uci-defaults/hotplug →
  áp `shell-script-review-skill` (quoting, TOCTOU, portability) +
  `openwrt-shell-domain-skill` (procd, ubus, UCI API, /lib/functions.sh)

**Bước 2: Check Makefile**
- `PKG_HASH` có match tarball thật không (nếu skill extract đã verify)
- `PKG_SOURCE_URL` có còn reachable không (WARNING nếu mirror sunset)
- `PKG_LICENSE` có set không, có đúng SPDX không
- `DEPENDS` có include đủ library dùng trong source không
- `define Package/<name>/{description,conffiles,postinst,prerm}` có hợp lý không
- `CONFIGURE_ARGS` / `MAKE_FLAGS` có leak host path, có `-j` hardcode không
- `$(call Build/Compile/Default)` vs custom — nếu custom phải check kỹ

**Bước 3: Check patches**
- Patch có header (`From`, `Subject`, `Signed-off-by`) không
- Patch có số thứ tự đúng format (`NNN-description.patch`) không
- Patch áp sạch lên `PKG_VERSION` hiện tại không (kiểm qua skill)
- Patch có upstream được không — có reference tới upstream PR/commit không
- Patch có conflict với nhau trong series không
- Patch sửa autoconf/Makefile → WARNING (fragile khi version bump)

**Bước 4: Check overlay & base-files shell**
- `init.d/*`: `USE_PROCD=1`, `START=/STOP=` priority, `service_triggers()`,
  `procd_open_instance` + `procd_close_instance`, respawn param order
- `uci-defaults/*`: chạy 1 lần, `exit 0`, không gọi service sớm, idempotent nếu có thể
- `hotplug.d/**`: guard `$ACTION` + `$INTERFACE`, không block
- `/lib/functions.sh` / `/lib/functions/*.sh`: function prefix namespace, không breakage
  tương thích với package khác đang source file này
- `/lib/upgrade/*.sh` và `sysupgrade` hooks: CRITICAL — sai có thể brick device
- Shell script nói chung: `shell-script-review-skill` §4-7 (quoting / error / TOCTOU / injection) +
  `openwrt-shell-domain-skill` §4-11 (OpenWrt API)

**Bước 5: Cross-check với source**
- `DEPENDS` vs `#include`: thiếu library → compile fail trên clean build
- `conffiles` list khớp với files/etc/config/ không
- `postinst` / `prerm` có handle đúng upgrade scenario

**Bước 6: Output findings + aggregate**

## 5. DOMAIN CHECKLIST

### Makefile
- [ ] `PKG_NAME` / `PKG_VERSION` / `PKG_RELEASE` có đủ
- [ ] `PKG_HASH` = SHA256 (không phải MD5 legacy)
- [ ] `PKG_SOURCE_URL` có fallback (`@GNU/` / `@APACHE/` / direct)
- [ ] `PKG_LICENSE` hợp lệ SPDX
- [ ] `PKG_MAINTAINER` set
- [ ] `DEPENDS` đủ và không thừa
- [ ] `define Package/<name>` có `SECTION`, `CATEGORY`, `TITLE`, `URL`
- [ ] `CONFIGURE_ARGS` không leak build host path
- [ ] `install` section không `chmod 777`, không hardcode UID

### Patches
- [ ] Tên file `NNN-description.patch` (N = 3 digits)
- [ ] Header `From:` `Subject:` `Signed-off-by:` (upstream-ready)
- [ ] Apply sạch (không fuzzy > 0) lên PKG_VERSION hiện tại
- [ ] Không patch file generated (Makefile.in, configure)
- [ ] Có reference upstream nếu đã merge upstream
- [ ] Không overlap nội dung giữa các patch

### Init scripts
- [ ] `START=` / `STOP=` priority hợp lý (50-99 cho user service)
- [ ] `USE_PROCD=1` hoặc sysv tùy convention
- [ ] `service_triggers()` reload config
- [ ] Không hardcode path binary (dùng `$PROG`)
- [ ] Error path có log rõ

### uci-defaults
- [ ] Script exit 0 sau khi chạy thành công
- [ ] Không rerun khi đã apply (hoặc idempotent)
- [ ] Không phụ thuộc service đã chạy

### Hotplug
- [ ] Check `$ACTION` / `$INTERFACE` / `$DEVICE` trước khi dùng
- [ ] Không block (no sleep loop, no network call dài)

## 6. OUTPUT FORMAT
Theo `review-findings.schema.json`. Mỗi finding include:
- `category`: `security` | `correctness` | `reliability` | `compat` | `style`
- `file`: path git thực (Makefile/patches/files/...), KHÔNG dùng virtual_source
  (virtual_source chỉ cho file trong tarball)

## 7. DEPENDENCIES
- `skills/domains/openwrt-build-domain-skill.md` — Makefile/patches/init/uci-defaults patterns
- `skills/domains/openwrt-shell-domain-skill.md` — procd/ubus/UCI/base-files API
- `skills/core/openwrt-source-context-skill.md` — verify patches apply, source context
- `skills/core/shell-script-review-skill.md` — shell generic (quoting, TOCTOU, portability)

## 8. FINAL CHECK
□ Đã review đủ 3 nhóm: Makefile / patches / files/?
□ Đã cross-check DEPENDS vs source include?
□ Đã verify patches apply sạch (không có `patches_failed`)?
□ Findings dùng path git, không virtual path?
□ Output đúng schema?
