---
name: review-orchestrator
description: >
  Skill này nên được dùng khi cần điều phối hoạt động code review trong hệ thống multi-agent,
  ví dụ: "review PR", "review code", "review commit gần nhất", "review 3 commit mới nhất",
  "review commit thứ 2 gần nhất", "review commit <hash>", "review module X",
  "analyze git diff", "impact analysis before merge".
  Output là review plan, phân rã task theo domain, mapping tới sub-agent, và tổng hợp findings.
  KHÔNG dùng cho: review chi tiết từng dòng code, debug root cause sâu, viết fix code.
version: 0.1.2
---

# SKILL: Review Orchestrator

<!--
Changelog:
  0.1.2 (2026-06-14):
    - Bước 2.0: thu hẹp rule cho file `.user` — chỉ trigger prefilter khi path
      khớp `**/etc/udhcpc.user.d/*` (canonical udhcpc hook location), KHÔNG
      trigger cho mọi `**/*.user` (e.g. lighttpd.user là credential file, không
      phải shell). Closes RV-MOD-20260614-02 feedback issue #3.
    - Bước 7 (NEW): "Binary asset auto-exclude" — file extension thuộc
      `binary_asset_extensions` ở `agent-registry.yaml` được map ở Bước 2 nhưng
      auto-excluded khỏi agent task scope. Closes feedback issue #5.
    - Bước 7 (NEW): "HTML sampling guidance" — khi domain có > 20 file cùng
      extension, sample heuristic + remainder vào missing_context[]. Closes
      feedback issue #6.

  0.1.1 (2026-06-13):
    - Bước 2.b: thêm tie-break rule 4 "tail-segment specificity wins"
      (cho 2 domain agent cùng match, glob có literal khớp segment sâu hơn thắng).
    - Bước 2.0 (mới): "Shell-extension prefilter" — file `.sh`/init.d/uci-defaults/
      hotplug.d luôn ưu tiên route tới agent có `shell-script-review-skill`
      (tránh case domain agent với glob rộng "nuốt" shell file mà không có
      skill review shell).
-->


## 1. PURPOSE
Điều phối toàn bộ phase code review trong mô hình multi-agent cho embedded SDLC.

## 2. WHEN TO USE
✅ Dùng khi:
- "review PR", "review code", "review firmware"
- "review commit gần nhất" / "review HEAD"
- "review N commit gần nhất" (N ≤ 10, ví dụ "review 3 commit mới nhất")
- "review commit thứ K gần nhất" (K ≤ 10, ví dụ "review commit thứ 2 gần nhất")
- "review commit <hash>" (commit ID cụ thể)
- "review module X" / "review toàn bộ package Y"
- "analyze diff", "impact analysis before merge"

❌ KHÔNG dùng:
- Review chi tiết từng dòng code
- Viết patch/fix implementation
- Phân tích root cause sâu khi chưa có log/context
- Thiết kế kiến trúc tổng thể mới

## 3. REVIEW MODES

| Mode | Dùng khi | Input chính |
|------|---------|-------------|
| `diff-review` | Review PR/patch | git diff hoặc changed files list |
| `commit-review` | Review commit(s) | `commit_spec` (ref, count, aggregation) |
| `full-review` | Audit module / code cũ | Danh sách files |
| `module-review` | Review toàn bộ một module | `module_path` |

### 3.1 `commit-review` chi tiết
`commit_spec` gồm:
- `ref`: `HEAD` | `HEAD~N` | `<commit-hash>` (commit mới nhất trong range)
- `count`: 1 ≤ count ≤ 10 (số commit lùi về từ `ref`)
- `aggregation`: `combined` (mặc định) | `per-commit`

Ví dụ mapping:
| User yêu cầu | commit_spec |
|--------------|-------------|
| "review commit gần nhất" | `{ref: HEAD, count: 1}` |
| "review 3 commit gần nhất" | `{ref: HEAD, count: 3}` |
| "review commit thứ 2 gần nhất" | `{ref: HEAD~1, count: 1}` |
| "review commit abc123" | `{ref: abc123, count: 1}` |
| "review 2 commit từ abc123 trở về" | `{ref: abc123, count: 2}` |

**Validation bắt buộc:**
- `count` ∈ [1, 10], nếu > 10 → reject và yêu cầu user giảm
- `ref` phải resolve được (hash tồn tại, HEAD~N trong depth của repo)

## 4. EXPECTED INPUTS
**Bắt buộc:**
- Review request JSON/YAML theo schema
- Mode: `diff-review` | `commit-review` | `full-review` | `module-review`
- Một trong: git diff / commit_spec / module_path / changed files list

**Khuyến nghị:**
- Requirement / CR / bug context
- Design note
- Known constraints

## 5. WORKFLOW

### Bước 0: Context Extraction (cho commit-review và module-review)

**0.1 Git context.** Gọi `git-context-skill`:
- `commit-review`: resolve `commit_spec` → lấy diff + commits metadata + changed_files
- `module-review`: `git ls-files <module_path>` → lấy scope_files

Nếu validation fail (count > 10, hash không tồn tại, repo shallow) → return error cho user.

> ⚠️ **KHÔNG được lọc hoặc loại bỏ file khỏi scope_files bằng heuristic tại bước này**
> (ví dụ: "open-source", "upstream", "third-party", "examples"). Toàn bộ scope_files phải
> được đưa nguyên vẹn vào Bước 2 để domain mapping thuật toán quyết định — không phải
> orchestrator tự phán xét. Chỉ loại bỏ file khi `allowed_paths` glob KHÔNG match sau
> khi đã chạy đầy đủ Bước 2a→2b→2c.

**0.2 OpenWrt tarball detection.** Sau khi có `changed_files` (hoặc `scope_files`),
detect OpenWrt package pattern:

```
for each unique <pkg_dir> chứa file thay đổi:
  if <pkg_dir>/Makefile tồn tại VÀ chứa (PKG_SOURCE + PKG_HASH):
    mark <pkg_dir> là OpenWrt package
```

Với mỗi OpenWrt package phát hiện được:
- Gọi `openwrt-source-context-skill` với `(package_dir, commit_ref)`
- Nhận: `openwrt_package` metadata + synthetic `diff_content` + virtual `changed_files[]`
  (mỗi file có `virtual_source`)
- Merge vào context:
  - File git-level (Makefile/patches/files/) giữ path git → sẽ route tới `openwrt-build-review-agent`
  - File virtual (trong tarball) dùng virtual path → sẽ route tới domain agent tương ứng theo tên file

Nếu skill trả `warnings` (patch conflict, network fail, hash mismatch) → ghi vào
`review-plan.missing_context[]` để reviewer biết.

**Fallback khi network unavailable + no cache:** không abort — vẫn review Makefile +
patches + files/ qua `openwrt-build-review-agent`, log warning "source diff unavailable,
review build-level only".

### Bước 1: Change Understanding
- Đọc diff/changed_files/context
- Tóm tắt change intent (từ commit messages nếu có)
- Xác định domain chính/phụ

### Bước 2: Domain Mapping

Mỗi file được map tới **đúng một agent** qua quy trình 4 giai đoạn (2.0 → 2.a → 2.b → 2.c):

**2.0 — Shell-extension prefilter (NEW v0.1.1, narrowed in v0.1.2).**
Nếu file:
- Extension `.sh` (bao gồm cả `**/lib/functions*.sh`, `**/lib/upgrade/*.sh`), HOẶC
- Path khớp một trong: `**/etc/init.d/*`, `**/etc/uci-defaults/*`, `**/etc/hotplug.d/**`, `**/etc/udhcpc.user.d/*`, `**/usr/libexec/rpcd/*`, HOẶC
- Tên file kết thúc `.init` (OpenWrt init script — luôn là shell), HOẶC
- **(v0.1.2)** Tên file kết thúc `.user` **VÀ path khớp `**/etc/udhcpc.user.d/*`** (udhcpc hook — luôn shell)

→ ưu tiên route tới agent có `shell-script-review-skill` trong `required_skills`,
ngay cả khi domain agent (mesh, dhcp, pppoe, …) cũng match glob của họ.

**Lưu ý v0.1.2 cho `.user` extension:**
`.user` suffix một mình KHÔNG đủ để trigger prefilter — phải kèm path
udhcpc.user.d. Lý do: nhiều file không-shell dùng `.user` suffix theo convention
khác (e.g. `lighttpd.user` = credential store; `*.user.conf` = user override
config). Route theo content type, không phải mere extension.
File `.user` không match udhcpc path đi tiếp xuống 2.a/2.b/2.c bình thường.

Thứ tự ưu tiên trong 2.0 (3 tier, dừng khi tier có ≥1 candidate):

**Tier 1**: Domain agent có path match `allowed_paths` **VÀ** có `shell-script-review-skill`
trong `required_skills`. Khi nhiều agent ở tier 1, áp dụng tie-break của Bước 2.b
(tail-segment specificity → longest-glob → extension-qualified → domain>build) để
chọn winner.

**Tier 2**: `openwrt-build-review-agent` nếu path match `allowed_paths` của nó.

**Tier 3**: `generic-review-agent` (luôn có `shell-script-review-skill`).

File virtual (có `virtual_source` metadata) KHÔNG dùng 2.0 — đi vào 2.a.

**Ví dụ:**
- `package/vnptt-unify/files/icwmpd/etc/init.d/icwmpd`:
  - Tier 1 candidates: mesh (vnptt-unify match + skill), tr069 (icwmpd match + skill v0.1.1)
  - Tie-break: tr069 thắng (literal `icwmpd` ở depth 4 > `vnptt-unify` ở depth 2)
- `package/vnptt-unify/files/vnptt-monitor-backhaul/usr/sbin/vnptt_monitor_backhaul.sh`:
  - Tier 1: chỉ mesh (vnptt-unify match + skill); tr069 không match path này
  - mesh thắng
- `package/foo/files/etc/init.d/foo` (file generic không thuộc domain):
  - Tier 1: rỗng
  - Tier 2: openwrt-build match `**/etc/init.d/*` → thắng

**Lý do:** một số domain agent có glob rộng (vd mesh `package/**/vnptt-unify/**`)
"nuốt" cả init.d/uci-defaults nhưng `required_skills` không có shell-script-skill —
route tới agent đó dẫn tới review thiếu chất lượng. Prefilter này đảm bảo shell
file luôn về agent có skill phù hợp.

**2.a — Virtual file check (precedence cao cho file virtual).**
Nếu file có metadata `virtual_source` (từ openwrt-source-context-skill) → KHÔNG dùng
`allowed_paths` (vì virtual path không tồn tại trong git). Map theo thứ tự:
1. Basename match trigger keyword của agent (ví dụ `dhcp_client.c` → `dhcp`)
2. Extension match web (`.html/.js/.css/.lua`) → `web-review-agent`
3. Còn lại → `generic-review-agent` (giữ `virtual_source` trong finding)

**2.b — Path match cho file git thật.**
File chưa được route ở 2.0 hoặc 2.a. Duyệt toàn bộ agent (trừ `is_fallback: true`),
chọn agent có glob trong `allowed_paths` match file path. Tie-breaker khi nhiều
agent match (áp dụng tuần tự, dừng khi có winner):

1. **Tail-segment specificity wins (NEW v0.1.1)**: glob có literal name khớp segment
   sâu hơn (gần cuối path hơn) thắng.
   - Ví dụ: file `package/vnptt-unify/files/icwmpd/etc/init.d/icwmpd`
     - mesh `package/**/vnptt-unify/**`: literal `vnptt-unify` ở **segment 2**
     - tr069 `package/**/icwmpd/**`: literal `icwmpd` ở **segment 4** (sâu hơn)
     - → tr069 thắng (literal name khớp segment sâu hơn = ý định cụ thể hơn về domain)
   - Đếm depth của segment match: segment đầu là 1, đếm tăng dần theo `/`.
   - Khi 2 glob cùng depth → đi tiếp rule 2.

2. **Longest-glob wins** (literal char count): glob cụ thể hơn (nhiều ký tự literal,
   ít wildcard) thắng.
   - Ví dụ `dl/vnptt-unify-1.0/**/icwmpd/**` > `dl/vnptt-unify-1.0/**`.

3. **Extension-qualified wins** nếu vẫn tie: `**/*.lua` > `package/**/*`.

4. **Domain agent > build agent** nếu vẫn tie: `dhcp-review-agent` > `openwrt-build-review-agent`
   khi file C trong tarball có cả 2 match (hiếm — chỉ xảy ra khi overlay có source).

5. **Skill-capability check (NEW v0.1.1)**: nếu winner ở rule 1-4 KHÔNG có
   `required_skills` phù hợp với file (ví dụ file `.py` nhưng agent chỉ có
   embedded-c-code-review-skill), log warning vào `plan.json.routing_warnings[]`
   nhưng giữ assignment — agent có thể delegate hoặc trả về low-confidence finding.
   Không tự ý reroute trừ khi extension match rule 2.0.

6. Ghi lại vào `plan.json.routing_decisions[]` với lý do chọn (rule nào áp dụng,
   các agent candidate và literal-char-count / depth của mỗi cái).

**2.c — Fallback (thứ tự cuối).**
File chưa match agent nào ở 2.a/2.b:
1. Extension `.html/.js/.css/.lua` → `web-review-agent` (extension-based safety net)
2. Shell `.sh` trong `package/base-files/**`, `**/etc/init.d/*`, `**/etc/uci-defaults/*`,
   `**/etc/hotplug.d/**`, `**/lib/functions*.sh`, `**/lib/upgrade/*.sh` →
   `openwrt-build-review-agent` (áp `shell-script-review-skill` + `openwrt-shell-domain-skill`)
3. File `.sh` khác → `generic-review-agent` với `shell-script-review-skill`
4. File `.c/.h/.cpp/.hpp` → `generic-review-agent` với `embedded-c-code-review-skill`
5. Còn lại → `unclassified_files`, xử lý ở Bước 3

**Trigger keyword không dùng để chọn agent primary** — chỉ dùng:
- Tăng priority cho task nếu diff/commit message chứa trigger của domain nào đó
- Gợi ý domain khi virtual path ambiguous (2.a.1)

### Bước 3: Unclassified Handling
Các file không match domain chuyên biệt nào:
1. Thêm vào `unclassified_files` trong review plan
2. Assign `generic-review-agent` với scope = `unclassified_files`
3. Nếu có ≥ 3 files cùng một pattern/path chung (ví dụ `libs/firewall/...`):
   - Emit `suggested_new_agents` entry với `suggested_domain`, rationale, sample_files
   - Log warning: "Pattern <X> có thể warrant domain agent riêng"

```json
"suggested_new_agents": [
  {
    "suggested_domain": "firewall",
    "rationale": "3 files trong libs/firewall/ được review bởi generic agent",
    "sample_files": ["libs/firewall/rules.c", "libs/firewall/nat.c", "libs/firewall/forward.c"]
  }
]
```

### Bước 4: Strategy Selection
Chọn strategy phù hợp với mode:
- `commit-review` / `diff-review`: focus vào diff, regression risk
- `full-review` / `module-review`: scan toàn bộ logic, resources, patterns
- Optional focus tags: `security`, `performance`, `compatibility`, `state-machine`, `memory`

### Bước 5: Task Decomposition
Tạo task theo logic: protocol correctness, state transitions, config compat, lifecycle,
memory, security, test gaps.

Cho `commit-review` với `aggregation: per-commit`: tạo task riêng cho từng commit.

### Bước 6: Agent Selection
Chọn sub-agent từ registry. Mỗi task: task_id, domain, assigned_agent, scope_files,
objective, priority.
- Ưu tiên agent chuyên biệt trước fallback
- `generic-review-agent` chỉ dùng khi không agent nào khác match

### Bước 7: Scope Control
Cấp context tối thiểu cho từng agent.

**7.1 Binary asset auto-exclude (NEW v0.1.2).**
File extension thuộc `binary_asset_extensions` ở `agent-registry.yaml`
(`.png/.jpg/.jpeg/.gif/.bmp/.ico/.svg/.webp/.woff/.woff2/.ttf/.eot/.otf/.mp3/.mp4/.wav/.ogg/.pdf`)
được map ở Bước 2 (giữ audit trail) nhưng PHẢI excluded khỏi agent's effective
`scope_files` ở task payload.

Ghi vào `plan.json.routing_decisions[]`:
```json
{
  "rule": "Bước 7.1 binary-asset auto-exclude",
  "decision": "<N> binary asset(s) remain mapped to <agent> but excluded from task scope",
  "rationale": "Non-text — no review value at code level. Mapping preserved for audit trail.",
  "files_excluded": ["...", "..."]
}
```

Lý do: nhiều agent có `allowed_paths` glob rộng (e.g. `dl/**/webserver-1.0/**`)
sẽ "nuốt" cả 52 file ảnh dưới `pages/images/`. Agent đọc binary file lãng phí
context, không cho ra finding hữu ích.

**7.2 HTML/repeated-file sampling guidance (NEW v0.1.2).**
Khi domain agent có > 20 file cùng extension (e.g. 58 file `.html` trong web
module), thay vì pass toàn bộ vào single task:

Option A: **Sample heuristic**. Chọn:
- File security-critical (theo keyword): tên chứa `login`/`auth`/`account`/`password`/`upload`/`backup`/`restore`/`reset`/`reboot`/`firewall`/`firmware`
- Plus stratified sample 30-50% các file còn lại
- Pass tới agent với note explicit về sampling
- Append `missing_context[]` entry liệt kê file KHÔNG được sample

Option B: **Multi-task partition**. Chia thành ≤3 sub-task của cùng agent,
mỗi sub-task ≤20 file, scope theo functional area (auth/config/diagnostics).
Mỗi sub-task có `task_id` riêng và spawn parallel.

Default: Option A khi tổng số file 20-50, Option B khi > 50.

Ghi vào `routing_decisions[]` lý do chọn approach + danh sách file
sample/non-sample.

**7.3 Minified third-party libs.**
File matching `**/*.min.{js,css}` hoặc tên known-lib (`jquery*.js`,
`bootstrap*.js`, `d3*.js`, `lodash*.js`, `react*.js`, `vue*.js`, `angular*.js`,
`underscore*.js`, `moment*.js`, `chart*.js`, `canvasjs*.js`, `nlbw*.js`) —
KHÔNG deep-review content. Chỉ kiểm tra version-pin / SRI / known-CVE.

Agent task scope nên explicit liệt kê các file này là "out-of-scope deep review,
flag CVE if version detectable".

### Bước 8: Aggregation
Merge findings, deduplicate, group theo severity/domain.

### Bước 9: Gap Detection
Kiểm tra: thiếu test, backward compat, error handling, reload/restart impact, performance.

### Bước 10: Final Output
Ba phase, thực hiện tuần tự:

**10.1 — Build aggregated report** (theo `review-report.schema.json`):
- Merge findings từ tất cả sub-agents
- Tính `stats` (by severity / category / domain / file)
- Tính `verdict` theo logic:
  - ≥1 `critical` → `block`
  - ≥1 `major` + confidence `high` → `block`
  - có `major`/`minor` còn lại → `needs-fix`
  - chỉ `info` hoặc zero → `ship`
- Tính `top_findings` (5-10 id) theo score = `severity_weight × confidence_weight × category_weight` (chi tiết ở `review-report-render-skill` §8)
- Viết `summary` (TL;DR 3-5 dòng plain text)
- Gom `missing_tests`, `human_attention`, `risk_summary`, `suggested_new_agents`

**10.2 — Gọi `review-report-render-skill`** để:
- Sinh `review_id` theo format `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`
- Tạo thư mục `.review/<review_id>/`
- Ghi artifacts: `meta.json`, `plan.json`, `findings/<agent>.json`, `report.json`, `report.md`, `report.html`
- Cập nhật `.review/index.json`

**10.3 — Return** đường dẫn `report.md` + `report.html` + `review_id` cho user.

## 6. OUTPUT FORMAT

### 6.1 Artifacts per review
Thư mục chuẩn: `.review/<review_id>/`
```
.review/RV-CMT-20260418-01-abc123d/
├── meta.json          # id, mode, verdict, stats — cho index đọc nhanh
├── plan.json          # review-plan.schema.json
├── report.json        # review-report.schema.json (aggregated)
├── report.md          # bản render Markdown cho người đọc
├── report.html        # bản render HTML self-contained
└── findings/
    ├── <agent-1>.json # raw output mỗi agent (review-findings.schema.json)
    └── <agent-2>.json
```
`.review/index.json` — danh sách toàn bộ review đã chạy (sort desc theo `generated_at`).

### 6.2 Naming convention
**Review ID:** `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`
| Mode | Prefix | Shorthash |
|------|--------|-----------|
| diff-review   | DIFF | — |
| commit-review | CMT  | 7 ký tự hash của commit mới nhất trong range |
| full-review   | FULL | — |
| module-review | MOD  | — |

`SEQ` = 2-digit counter theo ngày (01, 02, ...).

**Tên file:** cố định — `report.json`, `report.md`, `report.html`, `plan.json`, `meta.json`, `findings/<agent-id>.json`. Không timestamp/space/dấu trong tên file.

### 6.3 Markdown report — thứ tự section cố định
Chi tiết template ở `review-report-render-skill` §5. Tóm tắt thứ tự:
1. Header (verdict badge + stats)
2. Scope (commits / module / files / focus)
3. TL;DR
4. Top Findings
5. Findings by Severity (critical → info, dùng `<details>`)
6. Findings by Domain (table)
7. Risk Summary
8. Missing Tests (checkbox list, link `related_findings`)
9. Human Attention
10. Suggested New Agents
11. Footer

**Ổn định thứ tự** là yêu cầu hard — downstream automation (CI comment, dashboard) sẽ grep section header.

### 6.4 HTML report
Self-contained: inline CSS, không external asset. Dark theme, responsive layout, collapse via `<details>`. Template đầy đủ ở `review-report-render-skill` §6.

### 6.5 Automation inputs
- `meta.json` — status gate cho CI (`verdict` field)
- `report.json` — nguồn duy nhất cho bot/dashboard; `report.md` và `.html` chỉ để người đọc
- `top_findings[]` — comment PR ngắn gọn
- `suggested_new_agents[]` — trigger job tạo agent mới

## 7. GUARDRAILS
- KHÔNG review chi tiết từng dòng code
- KHÔNG tự viết code fix
- KHÔNG assign agent ngoài registry
- PHẢI giới hạn scope của từng sub-agent
- PHẢI trả output đúng schema
- PHẢI validate `commit_spec.count` ≤ 10
- PHẢI dùng `git-context-skill` cho `commit-review` / `module-review`
- PHẢI assign `generic-review-agent` cho file không match domain nào (không bỏ qua)
- PHẢI nêu rõ confidence nếu context thiếu
- **KHÔNG lọc file khỏi scope bằng heuristic** ("open-source", "upstream", "third-party",
  "examples"…) trước khi chạy Bước 2 — domain mapping thuật toán là cơ quan duy nhất
  quyết định file nào được review; orchestrator không được override bằng phán xét thủ công

## 8. DEPENDENCIES
- `schemas/agent-registry.yaml`
- `schemas/review-plan.schema.json`
- `schemas/review-findings.schema.json`
- `schemas/review-report.schema.json`
- `skills/core/git-context-skill.md`
- `skills/core/openwrt-source-context-skill.md`
- `skills/core/shell-script-review-skill.md`
- `skills/core/review-domain-mapping-skill.md`
- `skills/core/review-aggregation-skill.md`
- `skills/core/review-risk-skill.md`
- `skills/core/review-report-render-skill.md`
- domain skills + sub-skills tương ứng

## 9. FINAL CHECK
□ Đã validate `commit_spec.count` ∈ [1, 10]?
□ Đã dùng git-context-skill nếu mode là commit-review / module-review?
□ Toàn bộ scope_files từ git-context-skill đã được đưa vào Bước 2 **không lọc trước**?
□ Đã detect OpenWrt package và gọi openwrt-source-context-skill (nếu có)?
□ Patches failed (nếu có) đã log vào missing_context?
□ Đã xác định domain chính/phụ?
□ File không match domain → đã assign generic-review-agent?
□ Có pattern lặp → đã emit suggested_new_agents?
□ Agent mapping hợp lệ theo registry?
□ Scope mỗi task hẹp và rõ?
□ File web (HTML/JS/CSS/Lua/CGI) đã assign cho web-review-agent?
□ Script shell (.sh, init.d, uci-defaults, hotplug) đã assign đúng agent + shell skills?
□ **(v0.1.2)** Binary asset (`.png/.jpg/...`) đã được excluded khỏi agent scope (giữ mapping ở routing_decisions)?
□ **(v0.1.2)** Khi >20 file cùng extension → đã apply sampling/partition theo Bước 7.2 và ghi rationale?
□ **(v0.1.2)** File `.user` không thuộc udhcpc.user.d KHÔNG bị nhầm route qua shell prefilter?
□ Đã có risk summary + missing coverage?
□ Output đúng schema?
□ Đã tính verdict + top_findings?
□ Đã gọi review-report-render-skill để ghi artifacts vào `.review/<review_id>/`?
□ review_id đúng format `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<hash>]`?
□ Đã có đủ `report.json`, `report.md`, `report.html`, `meta.json`, `plan.json`, `findings/`?
□ Đã update `.review/index.json`?
