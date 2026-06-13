---
name: embedded-c-code-review-skill
description: >
  Skill này nên được dùng khi review mã C/C++ trong embedded/Linux/OpenWrt,
  ví dụ: "review C code", "check memory handling", "review error path", "embedded code review".
  Output là checklist chung cho sub-agent. KHÔNG dùng cho: kiến trúc hệ thống cấp cao.
version: 0.1.0
---

# SKILL: Embedded C Code Review

## 1. PURPOSE
Checklist dùng chung cho review mã C/C++ embedded. Bắt buộc include trong
mọi agent review C code (DHCP, PPPoE, Mesh, WiFi, CGI C, generic).

## 2. CORE CHECKLIST

### 2.1 Pointer & Bounds
- [ ] NULL check sau malloc / alloc-like function
- [ ] NULL check cho input pointer trước khi dereference
- [ ] Array index trong bounds (`i < size`, không phải `<=`)
- [ ] `strcpy`/`strcat`/`sprintf` thay bằng `strncpy`/`snprintf` với size đúng
- [ ] Buffer size truyền đúng (không `sizeof(ptr)` thay vì `sizeof(array)`)

### 2.2 Integer handling
- [ ] Size arithmetic không overflow (`n * sizeof(T)`)
- [ ] Signed/unsigned mixing không gây surprise (negative → huge unsigned)
- [ ] `atoi`/`strtol` có validate kết quả (errno, endptr)
- [ ] Bit shift không exceed type width

### 2.3 Memory ownership
- [ ] Mọi alloc có free tương ứng trên mọi path (bao gồm error path)
- [ ] Không double-free (free xong set = NULL)
- [ ] Không use-after-free sau close/disconnect
- [ ] Ownership convention rõ (caller-frees hay callee-frees)

### 2.4 Timer / Callback lifecycle
- [ ] Timer cancel trước khi free struct chứa nó
- [ ] Callback không access freed memory
- [ ] Timer armed lại đúng sau reconnect/reload
- [ ] Async callback race với shutdown được handle

### 2.5 Error path
- [ ] Mọi `return -1` / error cleanup đủ resources
- [ ] goto cleanup pattern dùng nhất quán
- [ ] Error propagation rõ ràng, không nuốt error silently
- [ ] Log error đủ context để debug (nhưng không spam)

### 2.6 Restart / Reload safety
- [ ] State persist đúng qua daemon restart (file, uci commit)
- [ ] In-memory state reset đúng sau reload
- [ ] Không race giữa old và new instance

### 2.7 Hot path & Logging
- [ ] Không log verbose trong packet/interrupt path
- [ ] Không malloc trong hot path nếu có thể
- [ ] Không syscall dư thừa (stat, open/close)

### 2.8 Backward compatibility
- [ ] Config schema không break (UCI option name, default)
- [ ] API/ABI không break (struct layout, function signature)
- [ ] File format / protocol magic numbers giữ nguyên

### 2.9 Concurrency (nếu có)
- [ ] Lock acquire/release pair đủ trên mọi path
- [ ] Không dead lock (lock order nhất quán)
- [ ] Atomic ops dùng đúng memory ordering

## 3. SEVERITY MAPPING (default)

| Issue | Severity |
|-------|----------|
| NULL deref exploitable, UAF, double-free | critical |
| Buffer overflow, memory leak trong long-running path | critical |
| Missing NULL check với caller-provided input | major |
| Missing free trong error path (one-shot code) | major |
| Log verbose in hot path | minor |
| Style, naming | info |

## 4. DEPENDENCIES
Thường dùng cùng domain skill (pppoe-domain-skill, wifi-driver-domain-skill, v.v.)
hoặc standalone bởi `generic-review-agent`.
