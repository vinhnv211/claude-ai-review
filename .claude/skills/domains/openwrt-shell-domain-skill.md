---
name: openwrt-shell-domain-skill
description: >
  Domain skill cho shell script trong OpenWrt: /lib/functions.sh API, config_* helpers,
  procd, ubus, network.sh, base-files overlay. Dùng bởi openwrt-build-review-agent khi
  review shell trong package/base-files/** hoặc files/ overlay của package khác.
version: 0.1.0
---

# SKILL: OpenWrt Shell Domain

## 1. PURPOSE
OpenWrt dùng shell (BusyBox ash) rất nặng cho system init, config, hotplug. Base-files
package chứa phần lớn infrastructure này. Skill cover idioms, API contract, và bug
pattern riêng của OpenWrt shell.

## 2. SCOPE
Áp dụng cho:
- `package/base-files/files/**/*` (không extension, có shebang)
- `package/base-files/files/lib/functions.sh`, `lib/functions/*.sh`
- `package/base-files/files/etc/init.d/*`
- `package/base-files/files/etc/uci-defaults/*`
- `package/base-files/files/etc/hotplug.d/**/*`
- `package/base-files/files/sbin/*`, `usr/sbin/*`, `bin/*`
- Shell script trong `files/` overlay của package khác (nếu dùng OpenWrt API)

**Không** cover shell thuần POSIX không dùng OpenWrt API — xem `shell-script-review-skill`.

## 3. BUSYBOX ASH CONTEXT
OpenWrt shell chạy trên BusyBox `ash` (không phải bash). Hệ quả:
- `local` được hỗ trợ (không như dash pure)
- Array KHÔNG có
- `[[ ]]`, `<<<`, `$'...'` KHÔNG có
- `trap ERR` KHÔNG có (chỉ EXIT/INT/TERM/HUP)
- `{a..b}` brace expansion KHÔNG có
- `<()` process substitution KHÔNG có

Mọi bug pattern của `shell-script-review-skill` §3.1–3.3 áp dụng — nếu script dùng bashism
sẽ fail trên device.

## 4. `/lib/functions.sh` API

Source bằng `. /lib/functions.sh`. Cung cấp các nhóm:

### 4.1 Config helpers
| Function | Purpose | Common bug |
|----------|---------|------------|
| `config_load <file>` | Load UCI config | Gọi nhiều lần không reset → state bẩn |
| `config_foreach <func> <type> [args]` | Iterate section | `func` không định nghĩa → silent skip |
| `config_get <dst> <section> <option> [default]` | Get value | Không quote `$dst` nếu dùng ngay |
| `config_get_bool <dst> <section> <option> [default]` | Get bool | Default thiếu → empty var nếu option missing |
| `config_set <section> <option> <value>` | In-memory set (không commit) | Nhầm với `uci set` |
| `config_list_foreach <section> <option> <func>` | Iterate list option | `func` phải accept 1 arg = value |

**Bug pattern 4.1.a — Thiếu `config_load` trước `config_get` (MAJOR):**
```sh
. /lib/functions.sh
config_get ip myapp lan_ip     # chưa load → empty
```
**Fix:** `config_load myapp` trước khi `config_get`.

**Bug pattern 4.1.b — Default không set (MINOR):**
```sh
config_get enabled myapp main enabled
[ "$enabled" = "1" ] && start   # nếu option chưa set → $enabled rỗng → false
```
**Fix:** `config_get_bool enabled myapp main enabled 0`.

**Bug pattern 4.1.c — Modify config không commit (MAJOR):**
```sh
config_set main port 8080       # chỉ in-memory!
# Thiếu uci commit → không persist
```
**Fix:** Dùng `uci set` + `uci commit` cho persistent, hoặc hiểu rõ `config_set` chỉ
là shim runtime.

### 4.2 Utility functions
| Function | Purpose | Common bug |
|----------|---------|------------|
| `append <var> <value> [sep]` | Append vào space-separated var | Không quote khi expand |
| `list_contains <list> <item>` | Check membership | Item có space → false positive |
| `find_mtd_part <name>` / `find_mtd_index` | MTD lookup | Tên partition wrong → empty |
| `default_postinst` / `default_prerm` | Package hook defaults | Override mà không gọi default |

### 4.3 Module-specific libs
- `/lib/functions/network.sh` — `network_get_ipaddr`, `network_is_up`, `network_flush_cache`
- `/lib/functions/service.sh` — `service_check`, `service_stop`
- `/lib/functions/procd.sh` — procd helpers (xem §6)
- `/lib/functions/system.sh` — `board_name`, `get_mac_binary`

**Bug pattern 4.3 — Quên `network_flush_cache` (MAJOR):**
```sh
. /lib/functions/network.sh
network_get_ipaddr ip lan        # dùng cache cũ nếu IP vừa đổi
```
**Fix:** `network_flush_cache` trước khi `network_get_ipaddr` nếu cần giá trị live.

## 5. UCI PATTERN

### 5.1 `uci get` không `-q` làm script fail (MAJOR)
```sh
val=$(uci get foo.bar.baz)        # bar không tồn tại → stderr + exit 1 → pipefail trip
```
**Fix:** `uci -q get foo.bar.baz` hoặc default `$(uci -q get ... || echo default)`.

### 5.2 Missing `uci commit` (MAJOR)
`uci set/add/delete` chỉ ghi vào delta file `/tmp/.uci/*` — cần `uci commit <pkg>` để ghi xuống `/etc/config/`.

### 5.3 Commit toàn bộ thay vì scope (MINOR)
```sh
uci commit                        # commit tất cả pkg có delta → race với process khác
```
**Fix:** `uci commit <pkg>` — chỉ pkg mình đang sửa.

### 5.4 `uci batch` vs nhiều lệnh rời (MINOR performance)
Nhiều `uci set` liên tiếp → mỗi lần parse file. Dùng batch:
```sh
uci -q batch <<EOF
set myapp.main.enabled=1
set myapp.main.port=8080
commit myapp
EOF
```

### 5.5 Anonymous section index fragile (MAJOR)
```sh
uci set firewall.@zone[0].name=lan    # [0] đổi khi reorder → sai zone
```
**Fix:** Dùng named section hoặc query `uci show | grep` theo attribute.

## 6. PROCD PATTERN (/etc/init.d/)

### 6.1 Skeleton chuẩn
```sh
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=10

start_service() {
    . /lib/functions.sh
    config_load myapp
    config_get port main port 8080

    procd_open_instance
    procd_set_param command /usr/sbin/myapp --port "$port"
    procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_set_param file /etc/config/myapp
    procd_close_instance
}

service_triggers() {
    procd_add_reload_trigger "myapp"
}

reload_service() {
    stop
    start
}
```

### 6.2 Bug patterns
**6.2.a — Thiếu `USE_PROCD=1` (MAJOR):** init script sẽ dùng legacy rc → mất tính năng respawn/triggers.

**6.2.b — Thiếu `procd_close_instance` (MAJOR):** instance không register → service không start.

**6.2.c — `procd_add_reload_trigger` thiếu → reload UCI không hiệu lực (MAJOR):**
Config đổi nhưng `/etc/init.d/myapp reload_config` (auto gọi khi commit) không do anything.

**6.2.d — respawn tham số sai thứ tự (MAJOR):**
Đúng: `respawn <threshold> <timeout> <retry>`.
```sh
procd_set_param respawn 5 3600 5     # nhầm threshold=5s → flood respawn
```

**6.2.e — `procd_set_param file` thiếu → config change không trigger restart (MINOR).**

**6.2.f — Chạy binary background với `&` thay vì procd (MAJOR):**
```sh
start_service() {
    /usr/sbin/myapp &               # procd không quản lý → không auto-respawn
}
```
**Fix:** Luôn qua `procd_set_param command`.

## 7. UBUS PATTERN

### 7.1 `ubus call` không check exit code (MAJOR)
```sh
ubus call network.interface.lan status          # fail nếu lan chưa up
# không check → parse output rỗng → bug khó debug
```
**Fix:**
```sh
json=$(ubus call network.interface.lan status 2>/dev/null) || {
    logger -t myapp "lan interface not ready"
    return 1
}
```

### 7.2 Parse output bằng grep/sed thay vì jsonfilter (MAJOR)
```sh
ip=$(ubus call network.interface.lan status | grep ipv4-address | awk ...)
```
**Fix:** Dùng `jsonfilter`:
```sh
. /usr/share/libubox/jshn.sh
json_load "$(ubus call network.interface.lan status)"
json_get_var ip "ipv4-address"
```
Hoặc:
```sh
ip=$(ubus call network.interface.lan status | jsonfilter -e '@["ipv4-address"][0].address')
```

### 7.3 `ubus listen` block không timeout (MAJOR)
```sh
ubus listen network.interface     # chạy forever, nếu script không fork → hang
```
**Fix:** Dùng timeout hoặc background + `kill` theo điều kiện.

## 8. HOTPLUG (/etc/hotplug.d/)

### 8.1 Không check $ACTION / $INTERFACE (MAJOR)
```sh
# /etc/hotplug.d/iface/50-myapp
restart_myapp        # chạy cho mọi ifup/ifdown/update
```
**Fix:**
```sh
[ "$ACTION" = "ifup" ] || exit 0
[ "$INTERFACE" = "wan" ] || exit 0
```

### 8.2 Block dài (MAJOR)
Hotplug sync — block 10s delay cả network bring-up sequence.
**Fix:** Fork background hoặc enqueue vào procd service.

### 8.3 Retrigger infinite loop (MAJOR)
Hotplug script modify interface state → trigger lại hotplug.
**Fix:** Guard bằng state file hoặc `ifup` flag.

## 9. UCI-DEFAULTS (/etc/uci-defaults/)

Đã cover chính ở `openwrt-build-domain-skill` §6. Bổ sung:

### 9.1 Script không xóa sau run (MAJOR)
OpenWrt tự `rm` script sau khi exit 0. Nếu `exit 1` → giữ lại và retry boot sau → loop.

### 9.2 Dùng `service start` trước khi rootfs_data mount (MAJOR)
uci-defaults chạy rất sớm. Gọi `/etc/init.d/foo start` có thể fail vì service chưa enable.
**Fix:** Chỉ set UCI config + `uci commit`. Service sẽ start ở boot sequence bình thường.

### 9.3 Thiếu `/etc/init.d/foo enable` (MINOR)
Nếu package cần auto-start mà Makefile chưa `$(INSTALL_BIN) ... S95foo`:
```sh
/etc/init.d/foo enable
```

## 10. /etc/rc.local & sysupgrade

### 10.1 `/etc/rc.local` block (MAJOR)
Code block → boot finish bị trì hoãn.
**Fix:** Fork hoặc chuyển sang init.d.

### 10.2 `/lib/upgrade/keep.d/` vs `/etc/sysupgrade.conf`
Khi review package, check file persist qua sysupgrade đã khai báo đúng chưa:
- List file cần giữ: `/lib/upgrade/keep.d/<pkg>`
- Hook custom: `/lib/upgrade/<pkg>.sh` với `pkg_extract_stage2` etc.

**Bug 10.2:** Config file ở custom path (không thuộc `/etc/config/`) → mất sau upgrade vì không khai báo.

## 11. LOGGING

### 11.1 `echo` thay vì `logger` (MINOR)
Init/hotplug script chạy không tty → `echo` vào void.
**Fix:** `logger -t myapp -p daemon.info "message"`.

### 11.2 Log level không phù hợp (MINOR)
Mọi thứ `logger` không `-p` → default `daemon.notice` → spam syslog.

## 12. BASE-FILES SPECIFIC

### 12.1 Modify `/lib/functions.sh` (CRITICAL)
Mọi package dùng — thay đổi có thể break hàng trăm script khác.
**Check:** PR đụng `/lib/functions.sh` phải có test matrix rộng và justify rõ.

### 12.2 Thêm function không prefix namespace (MAJOR)
Function toàn cục → conflict với package khác source cùng file.
**Fix:** Prefix `__myapp_foo()`.

### 12.3 Đụng `/sbin/sysupgrade`, `/lib/upgrade/` (CRITICAL)
Sai = brick device khi upgrade. Cần test đầy đủ trước khi merge.

### 12.4 `/etc/profile`, `/etc/shinit`, `/etc/banner` (MINOR)
Ảnh hưởng UX console nhưng ít critical. Check không break SSH welcome.

## 13. SEVERITY GUIDELINE

| Bug | Severity |
|-----|----------|
| Đụng `/sbin/sysupgrade` hoặc `/lib/upgrade/` sai | critical |
| Modify `/lib/functions.sh` không test rộng | critical |
| procd không register instance (thiếu close_instance) | major |
| Chạy service `&` thay vì procd | major |
| Thiếu `procd_add_reload_trigger` | major |
| `uci get` không `-q` + không handle fail | major |
| Thiếu `uci commit <pkg>` | major |
| respawn tham số sai thứ tự | major |
| Anonymous section index hard-code | major |
| Hotplug không check $ACTION/$INTERFACE | major |
| uci-defaults start service sớm | major |
| Function global không prefix trong base-files | major |
| Bashism với shebang sh (đã nêu ở shell-script-review-skill) | major |
| `config_get_bool` thiếu default | minor |
| Parse ubus bằng grep thay jsonfilter | major |
| `echo` thay logger trong init | minor |
| Thiếu `procd_set_param file` | minor |

## 14. TEST GAPS TO FLAG

- Init script start/stop/restart/reload trên device thật?
- `uci commit` + reload → service pickup config mới?
- Hotplug trigger đúng scope (action + interface)?
- uci-defaults chạy thành công first boot? rerun idempotent?
- sysupgrade preserve file đúng list?
- Script compat với BusyBox ash (không bashism)?
- Package remove (`opkg remove`) cleanup sạch?

## 15. DEPENDENCIES
- `shell-script-review-skill` (generic shell patterns)
- `openwrt-build-domain-skill` (có overlap ở init/uci-defaults/hotplug — skill này đi sâu hơn về API)
- Kiến thức: OpenWrt procd docs, UCI docs, ubus docs
