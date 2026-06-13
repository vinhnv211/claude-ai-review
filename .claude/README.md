# AI Multi-Agent Review Framework for Embedded SDLC

**Version:** 0.1.0 (first stable) · **Focus:** Embedded C + Shell + WebGUI
(OpenWrt router / Wi-Fi AP / camera IP)

Framework review mã nguồn đa agent cho các dự án embedded. Phạm vi cover:
- Code C/C++ (source chính + kernel driver)
- Shell (POSIX/ash/bash, OpenWrt idioms: procd/ubus/UCI)
- WebGUI frontend + backend (HTML/JS/CSS/Lua/LuCI/CGI)
- OpenWrt build infrastructure (Makefile/patches/overlay) và tarball source

📘 **Tài liệu kỹ thuật đầy đủ (HTML):** [`docs/index.html`](docs/index.html)

### Mục lục
1. [Mục tiêu & Phạm vi](#1-mục-tiêu--phạm-vi)
2. [Cấu trúc thư mục](#2-cấu-trúc-thư-mục)
3. [Kiến trúc tổng thể](#3-kiến-trúc-tổng-thể)
4. [Review Modes](#4-review-modes)
5. [Xử lý code không thuộc domain nào](#5-xử-lý-code-không-thuộc-domain-nào)
6. [Ví dụ Workflow](#6-ví-dụ-workflow)
7. [Output & Artifacts](#7-output--artifacts)
8. [Nguyên tắc mở rộng](#8-nguyên-tắc-mở-rộng)
9. [Contracts bắt buộc](#9-contracts-bắt-buộc)
10. [OpenWrt tarball source](#10-openwrt-tarball-source)
11. [Shell script review](#11-shell-script-review)
12. [Routing & tie-breaker rules](#12-routing--tie-breaker-rules)
13. [Roadmap](#13-roadmap-gợi-ý)

---

## 1. Mục tiêu & Phạm vi

### Framework hỗ trợ
- Review **commit** mới nhất, N commit gần nhất (1–10), commit thứ K, hoặc commit theo hash
- Review **module** toàn bộ (ví dụ `package/network/services/ppp`)
- Review **PR / diff** truyền thống
- Review **full audit** trước release
- Ngôn ngữ C / C++ (OpenWrt, embedded Linux, vendor SDK)
- WebGUI: HTML, JavaScript, CSS, LuCI / Lua, CGI (C / shell)
- 8 domain agents: DHCP, PPPoE, TR-069, Mesh, WiFi Driver, Web, OpenWrt Build, Generic (fallback)
- OpenWrt tarball source (dl/*.tar.gz) — auto extract + apply patches → review source thực
- Shell script (POSIX/ash/bash) + OpenWrt shell idioms (procd, ubus, uci, /lib/functions.sh)
- Không bỏ sót file — code không thuộc domain nào vẫn được review bởi fallback agent

### Ngoài phạm vi
- Review chi tiết từng dòng code
- Viết patch / fix / refactor tự động
- Debug root cause sâu
- Thiết kế kiến trúc hệ thống mới
- Phân tích git history dài hạn

---

## 2. Cấu trúc thư mục

```
ai-review-framework/
├── README.md                              Tổng quan framework
│
├── orchestrators/
│   ├── review-orchestrator.md             Orchestrator chính
│   └── review-extension-guide.md          Hướng dẫn mở rộng
│
├── agents/review/                         8 review agents
│   ├── dhcp-review-agent.md
│   ├── pppoe-review-agent.md
│   ├── tr069-review-agent.md
│   ├── mesh-review-agent.md
│   ├── wifi-driver-review-agent.md
│   ├── web-review-agent.md
│   ├── openwrt-build-review-agent.md      OpenWrt Makefile/patches/files
│   └── generic-review-agent.md            Fallback agent
│
├── skills/
│   ├── core/                              Skills dùng chung
│   │   ├── git-context-skill.md
│   │   ├── openwrt-source-context-skill.md  Extract tarball + apply patches
│   │   ├── embedded-c-code-review-skill.md
│   │   ├── shell-script-review-skill.md     Shell chung (POSIX/ash/bash)
│   │   ├── review-domain-mapping-skill.md
│   │   ├── review-aggregation-skill.md
│   │   ├── review-risk-skill.md
│   │   └── review-report-render-skill.md    Render JSON → MD/HTML
│   └── domains/                           Skills theo domain
│       ├── dhcp-domain-skill.md
│       ├── pppoe-domain-skill.md
│       ├── tr069-domain-skill.md
│       ├── mesh-domain-skill.md
│       ├── wifi-driver-domain-skill.md
│       ├── web-domain-skill.md
│       ├── web-html-skill.md              Web sub-skill
│       ├── web-javascript-skill.md        Web sub-skill
│       ├── web-cgi-skill.md               Web sub-skill
│       ├── web-luci-skill.md              Web sub-skill
│       ├── openwrt-build-domain-skill.md  OpenWrt build patterns
│       └── openwrt-shell-domain-skill.md  OpenWrt shell (procd/ubus/uci API)
│
├── schemas/                               JSON/YAML contracts
│   ├── agent-registry.yaml
│   ├── review-task.schema.json
│   ├── review-plan.schema.json
│   ├── review-findings.schema.json        Per-agent output
│   └── review-report.schema.json          Aggregated report (input cho render)
│
├── examples/                              Input/output mẫu
│   ├── review-request.example.json            diff-review
│   ├── review-request-commit.example.json     commit-review
│   ├── review-request-module.example.json     module-review
│   ├── review-plan.example.json
│   ├── review-findings.example.json
│   ├── review-findings-generic.example.json
│   ├── review-report.example.json             Aggregated report
│   ├── review-report.example.md               Render Markdown mẫu
│   └── review-report.example.html             Render HTML mẫu
│
└── docs/
    └── index.html                         Tài liệu kỹ thuật
```

---

## 3. Kiến trúc tổng thể

Framework tổ chức thành 5 tầng. 8 agent chia 3 nhóm theo vai trò: **domain** (review
source protocol), **cross-layer** (web, openwrt-build), **fallback** (generic).

```
TẦNG 1 — USER / CI
    "review commit gần nhất" | "review 3 commit" | "review module X" | "review PR"
    │
    ▼
TẦNG 2 — CONTEXT EXTRACTION  (Bước 0)
    ┌─ 0.1 git-context-skill ──────────────────────────────────────┐
    │    resolve commit_spec → diff + changed_files                │
    └─ 0.2 openwrt-source-context-skill (nếu có PKG_SOURCE+HASH) ──┘
         fetch tarball → verify SHA256 → apply patches →
         synthetic diff + virtual path
    │
    ▼
TẦNG 3 — REVIEW ORCHESTRATOR  (10 bước)
    1. Change Understanding       6. Agent Selection
    2. Domain Mapping             7. Scope Control
       2a virtual file routing    8. Aggregation
       2b path + longest-glob     9. Gap Detection
       2c fallback chain         10. Final Output (verdict + render)
    3. Unclassified Handling
    4. Strategy Selection         (ghi routing_decisions[] vào plan.json)
    5. Task Decomposition
    │
    ▼
TẦNG 4 — REVIEW AGENTS  (8 agents, cùng shape 8 section)
    Domain (source protocol)         Cross-layer            Fallback
    ├─ dhcp-review-agent             ├─ web-review-         └─ generic-
    ├─ pppoe-review-agent            │  agent                  review-agent
    ├─ tr069-review-agent            └─ openwrt-build-         (is_fallback)
    ├─ mesh-review-agent                review-agent
    └─ wifi-driver-review-agent         (Makefile+patches+
                                         base-files+shell)

    Mỗi agent: input = review-task, output = review-findings
    │
    │  ┌─ CORE SKILLS (shared) ──────────────────────────────┐
    │  │  embedded-c-code-review-skill                       │
    │  │  shell-script-review-skill                          │
    │  │  git-context-skill · openwrt-source-context-skill   │
    │  │  review-domain-mapping · -aggregation · -risk       │
    │  │  review-report-render-skill                         │
    │  ├─ DOMAIN SKILLS ─────────────────────────────────────┤
    │  │  dhcp · pppoe · tr069 · mesh · wifi-driver          │
    │  │  web-domain + web-html/js/cgi/luci                  │
    │  │  openwrt-build-domain · openwrt-shell-domain        │
    │  └─────────────────────────────────────────────────────┘
    ▼
TẦNG 5 — RENDER & OUTPUT  (Bước 10)
    review-report-render-skill:
    • merge findings → review-report.schema.json
    • verdict (block/needs-fix/ship) + top_findings ranking
    • render Markdown + HTML self-contained
    • ghi .review/<review_id>/ theo naming convention
    │
    ▼
ARTIFACTS — .review/<review_id>/
    meta.json · plan.json · report.json · report.md · report.html · findings/<agent>.json
```

**Phân loại 3 nhóm agent:**

| Nhóm | Agent | Review |
|------|-------|--------|
| Domain (source protocol) | dhcp, pppoe, tr069, mesh, wifi-driver | Source C trong package hoặc virtual path từ tarball |
| Cross-layer | web, openwrt-build | Multi-language (Web) hoặc build infrastructure + shell (OpenWrt build) |
| Fallback | generic | C/C++/shell không thuộc domain nào |

**Tại sao openwrt-build đứng riêng:** domain agent review *logic giao thức*; openwrt-build
review *vỏ bọc package* (Makefile, patches, init.d, /lib/functions.sh). Một commit có
thể đụng cả hai — openwrt-build review Makefile/patches/base-files, còn source trong
tarball được openwrt-source-context-skill sinh synthetic diff và route tới domain agent.
Hai nhánh chạy song song, ghép findings ở Bước 8.

**Tất cả 8 agent đóng shape giống nhau** — cùng skeleton 8 section (Purpose, When to use,
Expected inputs, Workflow, Domain checklist, Output format, Dependencies, Final check),
cùng contract input/output. Web và openwrt-build có thêm bước phân loại file nội bộ để
delegate tới sub-skills chuyên biệt (web: html/js/cgi/luci; openwrt-build: build-domain
/shell-domain/source-context), nhưng từ bên ngoài nhìn vào thì giống các agent khác.

---

## 4. Review Modes

| Mode | Dùng khi | Input chính |
|------|---------|-------------|
| `diff-review` | Review PR / patch | git diff hoặc changed_files |
| `commit-review` | Review 1–10 commit(s) | `commit_spec` (ref + count + aggregation) |
| `module-review` | Review toàn bộ module | `module_path` |
| `full-review` | Audit code cũ / pre-release | Danh sách files |

### Commit Selection (mode `commit-review`)

| User yêu cầu | `commit_spec` |
|--------------|---------------|
| "review commit gần nhất" | `{ref: "HEAD", count: 1}` |
| "review 3 commit gần nhất" | `{ref: "HEAD", count: 3}` |
| "review commit thứ 2 gần nhất" | `{ref: "HEAD~1", count: 1}` |
| "review commit abc123" | `{ref: "abc123", count: 1}` |
| "review 2 commit từ abc123" | `{ref: "abc123", count: 2}` |

**Giới hạn:**
- `count` phải ∈ [1, 10]. Yêu cầu &gt; 10 sẽ bị reject
- `ref` phải resolve được (hash tồn tại, HEAD~N trong depth của repo)
- `aggregation` = `combined` (mặc định, 1 diff tổng) hoặc `per-commit` (review từng commit)

---

## 5. Xử lý code không thuộc domain nào

Framework đảm bảo **mọi file đều có agent review**. Chiến lược 2 tầng:

**Tầng 1 — Fallback Agent.** `generic-review-agent` review mọi file C/C++ không match
domain chuyên biệt nào. Chỉ áp dụng checklist embedded C chung (null pointer, bounds,
memory, timer, error path).

**Tầng 2 — Gợi ý agent mới.** Nếu ≥ 3 files cùng pattern (cùng subdirectory hoặc keyword)
được fallback → orchestrator emit `suggested_new_agents`:

```json
"suggested_new_agents": [
  {
    "suggested_domain": "firewall",
    "rationale": "3 files trong libs/firewall/ được review bởi generic agent",
    "sample_files": ["libs/firewall/rules.c", "libs/firewall/nat.c"]
  }
]
```

Đây là tín hiệu cho team biết khi nào nên mở rộng framework với agent mới.

---

## 6. Ví dụ Workflow

### Review commit gần nhất
```
User: "review commit gần nhất"
 → commit_spec: {ref: HEAD, count: 1}
 → git show HEAD → changed_files
 → orchestrator map domain → assign sub-agent(s)
 → findings + risk + report
```

### Review 3 commit gần nhất
```
User: "review 3 commit mới nhất"
 → commit_spec: {ref: HEAD, count: 3, aggregation: combined}
 → git diff HEAD~3 HEAD → combined diff
 → orchestrator chia task theo domain (có thể gồm pppoe + web + generic)
 → parallel sub-agents → aggregated report
```

### Review module pppoe
```
User: "review toàn bộ module pppoe"
 → mode: module-review, module_path: package/network/services/ppp
 → git ls-files → 7 files (.c/.h)
 → orchestrator chọn full-review strategy
 → pppoe-review-agent audit toàn bộ module
 → report với legacy bug + test gaps
```

Xem chi tiết từng bước trong [`docs/index.html`](docs/index.html) → Ví dụ Workflow.

---

## 7. Output & Artifacts

Mỗi lần review ghi ra **một thư mục chuẩn hóa** dưới `.review/` ở repo root:

```
.review/
├── index.json                              Danh sách mọi review đã chạy
└── RV-CMT-20260418-01-abc123d/             Một review
    ├── meta.json                           id, mode, verdict, stats (cho CI gate)
    ├── plan.json                           Review plan (domain mapping, tasks)
    ├── report.json                         Aggregated report (canonical source)
    ├── report.md                           Render Markdown cho người đọc / PR comment
    ├── report.html                         Render HTML self-contained (inline CSS)
    └── findings/
        ├── dhcp-review-agent.json          Raw per-agent findings
        └── web-review-agent.json
```

### Review ID format
`RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`

| Mode | Prefix | Shorthash |
|------|--------|-----------|
| diff-review   | `DIFF` | — |
| commit-review | `CMT`  | 7 ký tự hash của commit mới nhất trong range |
| full-review   | `FULL` | — |
| module-review | `MOD`  | — |

### Verdict (auto-compute từ stats)
- `block` — có `critical`, hoặc `major` với confidence `high`
- `needs-fix` — có `major` / `minor` (không rơi vào block)
- `ship` — chỉ `info` hoặc zero findings

### Cho automation
- **`meta.json`** — field `verdict` dùng làm CI gate
- **`report.json`** — source of truth cho bot/dashboard
- **`top_findings[]`** — nội dung comment gọn cho PR
- **`suggested_new_agents[]`** — trigger workflow tạo agent mới

Chi tiết template MD/HTML + ranking algorithm: [`skills/core/review-report-render-skill.md`](skills/core/review-report-render-skill.md).

---

## 8. Nguyên tắc mở rộng

### Thêm Agent mới (4 bước)
1. Thêm entry vào `schemas/agent-registry.yaml`
2. Tạo `agents/review/<domain>-review-agent.md` (8-section skeleton)
3. Tạo `skills/domains/<domain>-domain-skill.md` (bug patterns + code examples)
4. (Tùy chọn) Tách sub-skills khi domain phức tạp — xem pattern `web-*`

### Thêm Skill mới
- **Core skill** (`skills/core/`): shared logic, thêm vào `required_skills` của agents cần dùng
- **Domain skill** (`skills/domains/`): knowledge chuyên cho 1 domain, gắn với agent tương ứng
- **Sub-skill**: tách từ domain skill khi cần chi tiết nhiều layer

### Không cần sửa Orchestrator
Orchestrator chỉ đọc registry + schema. Agent/skill mới tuân thủ contract → hoạt động ngay.

📖 Chi tiết: [`orchestrators/review-extension-guide.md`](orchestrators/review-extension-guide.md)

---

## 9. Contracts bắt buộc

Mọi agent phải:
- Nhận input theo [`review-task.schema.json`](schemas/review-task.schema.json)
- Chỉ review trong `scope_files` được cấp (không đọc toàn repo)
- Trả output theo [`review-findings.schema.json`](schemas/review-findings.schema.json)
- Severity ∈ `{critical, major, minor, info}`
- Confidence ∈ `{high, medium, low}`
- Mỗi finding có đủ: `evidence`, `impact`, `recommendation`, `confidence`

---

## 10. OpenWrt tarball source

OpenWrt package thường có source trong `dl/*.tar.gz`, git chỉ track Makefile + patches +
overlay. Framework auto-detect pattern này và xử lý:

```
Commit đụng Makefile (bump PKG_VERSION + PKG_HASH)
    → openwrt-source-context-skill fetch + verify hash + extract cả 2 tarball
    → apply patches
    → generate synthetic diff (tarball cũ vs tarball mới)
    → orchestrator route diff theo tên file thực trong tarball
          ├─ src/dhcp_client.c  → dhcp-review-agent
          ├─ lua/apps.lua       → web-review-agent
          └─ util/helper.c      → generic-review-agent
    → file git-level (Makefile/patches/files) → openwrt-build-review-agent
```

Mỗi finding trong tarball có `virtual_source` metadata (package, version, via_patch,
tarball_url) để reviewer reproduce.

**Chi tiết:** [`skills/core/openwrt-source-context-skill.md`](skills/core/openwrt-source-context-skill.md)

---

## 11. Shell script review

Shell rất phổ biến trong OpenWrt (base-files, init.d, uci-defaults, hotplug). Framework
cover 2 layer:

**Layer generic — `shell-script-review-skill`** (core):
- Quoting / word splitting / IFS
- `set -e` / pipefail pitfalls
- TOCTOU, symlink attack, `rm -rf "$var/"` khi var rỗng
- Injection (`eval`, command substitution)
- Bashism vs POSIX/ash portability
- Cross-reference với ShellCheck rules

**Layer OpenWrt — `openwrt-shell-domain-skill`** (domain):
- `/lib/functions.sh` API: `config_load`, `config_foreach`, `config_get_bool`
- UCI: `uci -q get`, `uci commit <pkg>`, `uci batch`, anonymous index
- procd: `USE_PROCD=1`, instance lifecycle, `respawn` param order, reload trigger
- ubus: `jsonfilter` / `jshn.sh` thay vì grep parse
- Hotplug: `$ACTION` / `$INTERFACE` guard, không block
- sysupgrade: `/lib/upgrade/keep.d/`, critical paths

**Routing:**
```
package/base-files/**       → openwrt-build-review-agent
**/etc/init.d/*             → openwrt-build-review-agent
**/etc/uci-defaults/*       → openwrt-build-review-agent
**/etc/hotplug.d/**         → openwrt-build-review-agent
**/lib/functions*.sh        → openwrt-build-review-agent
**/lib/upgrade/*.sh         → openwrt-build-review-agent
**/*.sh (ngoài các path trên) → generic-review-agent + shell-script-review-skill
```

---

## 12. Routing & tie-breaker rules

Khi một file match nhiều agent, orchestrator áp rule sau (theo thứ tự ưu tiên cao →
thấp) — chi tiết ở `orchestrators/review-orchestrator.md` Bước 2:

### 12.1 Virtual files (tarball extract)
File có metadata `virtual_source` KHÔNG match qua `allowed_paths` (virtual path không
tồn tại trong git). Map theo:
1. Basename match trigger keyword của domain agent (`dhcp_client.c` → `dhcp`)
2. Extension web (`.html/.js/.css/.lua`) → `web-review-agent`
3. Còn lại → `generic-review-agent` (giữ `virtual_source` để trace)

### 12.2 File git thật
Duyệt agent có `is_fallback: false`, chọn agent match qua `allowed_paths`. Khi nhiều
agent match cùng lúc:
1. **Longest-glob wins** — glob cụ thể hơn thắng.
   VD: `dl/vnptt-unify-1.0/**/icwmpd/**` (tr069) > `dl/vnptt-unify-1.0/**` (mesh)
2. **Extension-qualified wins** nếu vẫn tie: `**/*.lua` > `package/**/*`
3. **Domain agent > build agent** nếu vẫn tie

Kết quả ghi vào `plan.json.routing_decisions[]` để audit.

### 12.3 Fallback
File chưa match:
- Web extension → `web-review-agent`
- Shell `.sh` trong OpenWrt path (base-files, init.d, uci-defaults, hotplug,
  `/lib/functions*.sh`, `/lib/upgrade/*.sh`) → `openwrt-build-review-agent`
- Shell `.sh` khác → `generic-review-agent` + `shell-script-review-skill`
- C/C++ (`.c/.h/.cpp/.hpp`) → `generic-review-agent` + `embedded-c-code-review-skill`
- Còn lại → `unclassified_files` (log + check suggested_new_agents)

---

## 13. Roadmap gợi ý

Các domain agent có thể bổ sung cho embedded router / camera:
- `firewall-review-agent` — iptables / nftables / fw3 / fw4
- `nat-review-agent` — NAT / routing
- `camera-review-agent` — RTSP / v4l2 / ISP / encode
- `openwrt-git-source-skill` — xử lý `PKG_SOURCE_PROTO:=git` (hiện chưa có)

Khi `generic-review-agent` liên tục report pattern tương tự cho một nhóm file, đó là tín
hiệu đáng tin cậy nên thêm domain agent mới.

---

## Phiên bản & trạng thái

- **v0.1.0** — bản phát hành đầu tiên, logic đầy đủ:
  - 8 review agents (dhcp, pppoe, tr069, mesh, wifi-driver, web, openwrt-build, generic)
  - 7 core skills (git-context, openwrt-source-context, embedded-c-code-review,
    shell-script-review, review-domain-mapping, review-aggregation, review-risk,
    review-report-render)
  - 11 domain skills (5 protocol + 5 web + 2 OpenWrt shell/build)
  - 5 schemas (agent-registry, review-task, review-plan, review-findings, review-report)
  - 4 review modes (diff / commit / module / full)
  - 3 output formats (JSON canonical / Markdown / HTML)
  - OpenWrt tarball source + synthetic diff + virtual path routing
  - Routing tie-breaker rules (longest-glob, is_fallback last, virtual bypass)

**Paths trong `schemas/agent-registry.yaml`** hiện được tuỳ biến cho dự án cụ thể
(vnptt-unify, icwmpd, MT7916…). Khi áp dụng framework cho repo khác, sửa `allowed_paths`
phù hợp — không đụng các file contract khác.
