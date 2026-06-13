---
name: review-domain-mapping-skill
description: >
  Skill này nên được dùng khi cần map code change sang domain review phù hợp,
  ví dụ: "map changed files to domain", "determine affected module", "identify review domain".
  Output là affected domains, candidate agents, và danh sách unclassified files.
  KHÔNG dùng cho: review code chi tiết.
version: 0.1.1
---

# SKILL: Review Domain Mapping

<!--
Changelog:
  0.1.1 (2026-06-13):
    - Bước 1 (Path match): bổ sung tie-break "tail-segment specificity wins"
      và "skill-capability check" — đồng bộ với review-orchestrator.md v0.1.1.
    - Bước 0 (mới): Shell-extension prefilter chạy trước Bước 1 — file shell
      luôn được ưu tiên route tới agent có shell-script-review-skill.
-->


## 1. PURPOSE
Map change sang domain thực sự bị ảnh hưởng dựa trên path + logic impact + extension.
Xác định agent phù hợp từ `agent-registry.yaml`, và gom các file không match vào
nhóm `unclassified_files` để route sang fallback agent.

## 2. PRINCIPLES
- Logic impact quan trọng hơn tên file/path
- 1 change có thể map nhiều domain
- Luôn xét shared integration points: config, lifecycle, service restart, platform hooks
- File không match không được bỏ qua — phải đi vào fallback (`generic-review-agent`)

## 3. MAPPING ALGORITHM

Với mỗi file trong changed_files hoặc scope_files, thực hiện theo thứ tự:

### Bước 0: Shell-extension prefilter (v0.1.1, precedence cao nhất cho file git thật)
Nếu file thuộc một trong:
- Extension `.sh` (gồm `**/lib/functions*.sh`, `**/lib/upgrade/*.sh`)
- Path khớp: `**/etc/init.d/*`, `**/etc/uci-defaults/*`, `**/etc/hotplug.d/**`,
  `**/etc/udhcpc.user.d/*`, `**/usr/libexec/rpcd/*`
- Tên file kết thúc `.init` hoặc `.user`

→ Ưu tiên route theo thứ tự:
1. Agent domain match (path glob hit) **VÀ** có `shell-script-review-skill` trong `required_skills`
2. `openwrt-build-review-agent` nếu path match `allowed_paths` của nó
3. `generic-review-agent`

File có `virtual_source` metadata KHÔNG dùng Bước 0 — bypass sang Bước 1.

### Bước 1: Path match
Kiểm tra `file_path` có khớp với bất kỳ `allowed_paths` glob nào trong registry không.
- Khớp → candidate agent. Khi nhiều agent match, dùng tie-break (xem dưới).
- Không → Bước 2

**Tie-break (v0.1.1, áp dụng tuần tự):**

| # | Rule | Ý nghĩa |
|---|------|---------|
| 1 | **Tail-segment specificity wins** | Glob có literal name khớp segment sâu hơn của path thắng. Ví dụ với `package/vnptt-unify/files/icwmpd/etc/init.d/icwmpd`: mesh `package/**/vnptt-unify/**` literal ở segment 2 vs tr069 `package/**/icwmpd/**` literal ở segment 4 → tr069 thắng. |
| 2 | **Longest-glob wins** | Glob có nhiều ký tự literal hơn (ít wildcard) thắng. Ví dụ `dl/vnptt-unify-1.0/**/icwmpd/**` > `dl/vnptt-unify-1.0/**`. |
| 3 | **Extension-qualified wins** | `**/*.lua` > `package/**/*`. |
| 4 | **Domain agent > build agent** | `dhcp-review-agent` > `openwrt-build-review-agent` cho file C trong tarball overlay. |
| 5 | **Skill-capability check** | Nếu winner không có skill phù hợp cho extension của file → log `routing_warnings[]` nhưng giữ assignment (không reroute trừ khi rule Bước 0 áp dụng). |

### Bước 2: Trigger match
Nếu commit message hoặc diff content có chứa `triggers` keyword của agent nào.
- Khớp → candidate agent tương ứng
- Không → Bước 3

> Lưu ý: Trigger keyword **không override** agent đã chọn ở Bước 0 hoặc Bước 1.
> Trigger chỉ dùng khi không có path/extension match nào.

### Bước 3: Extension match (cho web)
Nếu extension là `.html/.htm/.js/.css/.lua` → `web-review-agent`

### Bước 4: Fallback
File `.c/.h/.cpp/.hpp` không match ở bước 0-3 → `generic-review-agent`
File khác (không phải C/web/shell) không match → flag trong `missing_context`

## 4. DETECTING REPEATED UNCLASSIFIED PATTERNS

Sau khi map xong, nếu `unclassified_files` có pattern lặp lại, emit suggestion:

**Tiêu chí phát hiện pattern:**
- ≥ 3 files trong cùng subdirectory (ví dụ `libs/firewall/`)
- Hoặc ≥ 3 files chứa cùng keyword trong tên (ví dụ tất cả có `fw_` prefix)

**Output:**
```json
"suggested_new_agents": [
  {
    "suggested_domain": "firewall",
    "rationale": "3 files trong libs/firewall/ được review bởi generic agent",
    "sample_files": ["libs/firewall/rules.c", "libs/firewall/nat.c", "libs/firewall/forward.c"]
  }
]
```

## 5. OUTPUT
```json
{
  "primary_domain": "pppoe",
  "secondary_domains": ["config-lifecycle"],
  "integration_domains": ["network"],
  "recommended_agents": [
    {"agent": "pppoe-review-agent", "scope_files": ["..."]},
    {"agent": "generic-review-agent", "scope_files": ["libs/utils/helpers.c"]}
  ],
  "unclassified_files": ["libs/utils/helpers.c"],
  "suggested_new_agents": []
}
```

## 6. EXAMPLES

### Example 1: Single domain match
Input: `package/network/services/ppp/pppoe.c`
→ Path match `package/**/ppp/**` → `pppoe-review-agent`

### Example 2: Multiple domains
Input: `package/network/services/ppp/pppoe.c`, `package/luci/app/wifi.lua`
→ pppoe.c → pppoe-review-agent
→ wifi.lua → web-review-agent (extension match)

### Example 3: Unclassified
Input: `libs/utils/string_helpers.c`
→ Không path/trigger match → `generic-review-agent`

### Example 4: Pattern suggest
Input: `libs/firewall/rules.c`, `libs/firewall/nat.c`, `libs/firewall/forward.c`
→ Tất cả unclassified → `generic-review-agent`
→ Emit suggestion tạo `firewall-review-agent`
