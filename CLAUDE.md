# CLAUDE.md — xsw240n_firmware

## Code review workflow (BẮT BUỘC)

Khi user yêu cầu review code (bất kỳ dạng nào):
- "review commit gần nhất", "review N commit gần nhất" (N ≤ 10)
- "review commit thứ K gần nhất", "review commit `<hash>`"
- "review module X", "review toàn bộ package Y"
- "review PR", "review diff", "analyze git diff"
- "impact analysis before merge", "check code quality", "audit code"

**PHẢI**: Dùng framework AI Review Multi-Agent tại `.claude/` — KHÔNG tự review bằng
general knowledge, KHÔNG dùng built-in skill `review` (dành cho GitHub PR).

### Entry point
Đọc `.claude/orchestrators/review-orchestrator.md` trước khi làm gì khác.

### Quy trình bắt buộc (10 bước)
- **Bước 0**: Context Extraction
  - 0.1 `git-context-skill` — resolve commit_spec / module files
  - 0.2 `openwrt-source-context-skill` — detect OpenWrt package (Makefile có
    `PKG_SOURCE` + `PKG_HASH`), fetch tarball, verify SHA256, apply patches,
    sinh synthetic diff + virtual path
- **Bước 1**: Change Understanding
- **Bước 2**: Domain Mapping (4 giai đoạn, v0.1.2)
  - **2.0** Shell-extension prefilter — file `.sh`/init.d/uci-defaults/hotplug.d
    luôn ưu tiên agent có `shell-script-review-skill`.
    **v0.1.2:** `.user` suffix CHỈ trigger prefilter khi path khớp
    `**/etc/udhcpc.user.d/*` (canonical udhcpc hook). Tránh false-trigger với
    file như `lighttpd.user` (credential file, không phải shell).
  - 2a Virtual files → basename trigger / extension (bypass allowed_paths)
  - 2b Git files → path match, tie-breaker (theo thứ tự):
    (1) **tail-segment specificity wins**, (2) longest-glob wins,
    (3) extension-qualified, (4) domain > build, (5) skill-capability check
  - 2c Fallback chain (web → openwrt-build shell → generic)
  - Ghi `routing_decisions[]` vào plan.json
- **Bước 3**: Unclassified Handling (≥3 files cùng pattern → `suggested_new_agents`)
- **Bước 4-5**: Strategy Selection + Task Decomposition
- **Bước 6**: Agent Selection — chỉ từ `.claude/schemas/agent-registry.yaml`
- **Bước 7**: Scope Control (scope tối thiểu cho mỗi agent)
  - **7.1 (v0.1.2)** Binary asset auto-exclude — file thuộc `binary_asset_extensions`
    (png/jpg/gif/svg/woff/ttf/mp3/pdf...) giữ mapping nhưng EXCLUDE khỏi
    agent task scope. Mục đích: tránh agent đọc binary file lãng phí context.
  - **7.2 (v0.1.2)** Sampling cho >20 file cùng extension — security-critical
    heuristic + stratified sample, hoặc multi-task partition (≤3 sub-task,
    ≤20 file/task). Remainder vào `missing_context[]`.
  - **7.3 (v0.1.2)** Minified third-party libs (`jquery*.js`, `*.min.js`,
    `bootstrap*`, `d3*`, `canvasjs*`...) out-of-scope deep review; chỉ
    version-pin / SRI / known-CVE check.
- **Bước 8**: Aggregation (merge + dedupe + **normalize**)
  - **v0.1.2** Pre-merge normalization PHẢI áp dụng:
    - `category` alias map (xem `agent-registry.yaml` `category_aliases`):
      `authentication`/`input-validation`/`csrf`/`xss` → `security`;
      `config` → `configuration`; `maintainability`/`documentation` → `style`
    - `missing_tests` plain-string → object (id/description/related_findings/priority)
    - Log normalization vào `routing_warnings[]`
- **Bước 9**: Gap Detection (missing tests, backward compat, performance)
- **Bước 10**: Final Output
  - Build `review-report.schema.json` (verdict + top_findings)
  - **v0.1.2** Apply **demojibake** pass cho mọi text field — fix double-UTF-8
    mojibake (em-dash `—` thành `â€"` v.v.) từ persisted-output pipeline.
    Xem `review-report-render-skill` §4.5.
  - Gọi `review-report-render-skill` → ghi artifacts
  - Return `review_id` + `report.md` + `report.html`

### 9 Review Agents hợp lệ (v0.1.1)
Chỉ assign agent có id trong `.claude/schemas/agent-registry.yaml`:
1. `dhcp-review-agent` — DHCP protocol/lease
2. `pppoe-review-agent` — PPP/PPPoE session
3. `tr069-review-agent` — CWMP/TR-181/BBF-DM (v0.1.1: cover `vnptt-cwmp`/`libbbfdm*` trong `dl/`)
4. `mesh-review-agent` — Mesh topology/backhaul (v0.1.1: thêm shell-script-skill)
5. `wifi-driver-review-agent` — cfg80211/mac80211
6. `web-review-agent` — WebGUI (HTML/JS/CSS/Lua/CGI)
7. `openwrt-build-review-agent` — Makefile/patches/files/base-files/shell/init.d/hotplug
8. `python-tools-review-agent` — **NEW v0.1.1**: Python build tools (.py, Dockerfile, code-gen)
9. `generic-review-agent` — fallback (C/C++/shell không thuộc domain nào)

### Output convention
- Thư mục: `.review/<review_id>/`
- Review ID: `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`
  - MODE: `DIFF` | `CMT` | `FULL` | `MOD`
  - Shorthash 7-char commit hash (chỉ CMT mode)
- Files cố định: `meta.json`, `plan.json`, `report.json`, `report.md`, `report.html`,
  `findings/<agent-id>.json`
- Cập nhật `.review/index.json` sau mỗi review

### Verdict auto-compute
- `block` — có `critical`, hoặc `major` + confidence `high`
- `needs-fix` — có `major`/`minor` (không rơi vào block)
- `ship` — chỉ `info` hoặc zero findings

## Cấu trúc framework

```
.claude/
├── orchestrators/
│   ├── review-orchestrator.md          ← Entry point (đọc đầu tiên)
│   └── review-extension-guide.md
├── agents/review/                       ← 9 domain agents (v0.1.1)
│   ├── dhcp-review-agent.md
│   ├── pppoe-review-agent.md
│   ├── tr069-review-agent.md
│   ├── mesh-review-agent.md
│   ├── wifi-driver-review-agent.md
│   ├── web-review-agent.md
│   ├── openwrt-build-review-agent.md
│   ├── python-tools-review-agent.md     ← NEW v0.1.1
│   └── generic-review-agent.md
├── skills/
│   ├── core/                           ← 8 core skills (shared)
│   │   ├── git-context-skill.md
│   │   ├── openwrt-source-context-skill.md  (Bước 0.2)
│   │   ├── embedded-c-code-review-skill.md
│   │   ├── shell-script-review-skill.md
│   │   ├── review-domain-mapping-skill.md
│   │   ├── review-aggregation-skill.md
│   │   ├── review-risk-skill.md
│   │   └── review-report-render-skill.md    (Bước 10)
│   └── domains/                         ← 13 domain skills (v0.1.1)
│       ├── dhcp-domain-skill.md
│       ├── pppoe-domain-skill.md
│       ├── tr069-domain-skill.md
│       ├── mesh-domain-skill.md
│       ├── wifi-driver-domain-skill.md
│       ├── web-domain-skill.md
│       ├── web-html-skill.md
│       ├── web-javascript-skill.md
│       ├── web-cgi-skill.md
│       ├── web-luci-skill.md
│       ├── openwrt-build-domain-skill.md
│       ├── openwrt-shell-domain-skill.md
│       └── python-tools-domain-skill.md ← NEW v0.1.1
├── schemas/                             ← JSON/YAML contracts
│   ├── agent-registry.yaml             (source of truth cho agent list)
│   ├── review-task.schema.json
│   ├── review-plan.schema.json
│   ├── review-findings.schema.json
│   └── review-report.schema.json
├── skills/review-orchestrator/SKILL.md  ← Auto-trigger wrapper
└── commands/review.md                   ← Slash command /review
```

## Guardrails tuyệt đối

- **KHÔNG** tự spawn agent trực tiếp mà không qua orchestrator
- **KHÔNG** dùng built-in `/review` hoặc `/security-review` của Claude Code cho
  review code internal (những slash command đó cho GitHub PR)
- **KHÔNG** assign agent ngoài `.claude/schemas/agent-registry.yaml`
- **KHÔNG** skip Bước 0.2 (OpenWrt tarball detection) nếu repo có
  `package/**/Makefile` — BẮT BUỘC kiểm tra `PKG_SOURCE` + `PKG_HASH`.
  v0.1.1: nếu phát hiện **Pattern C "pseudo-tarball in-tree"** (PKG_SOURCE có
  nhưng PKG_HASH vắng), KHÔNG abort — treat files trong `dl/<name>-<ver>/`
  như git-tracked, emit warning vào `missing_context[]`.
- **KHÔNG** skip Bước 10 (render MD/HTML + ghi artifacts). Không có output file =
  review chưa hoàn thành.
- **PHẢI** validate `commit_spec.count ∈ [1, 10]`. Yêu cầu > 10 → reject
- **PHẢI** trả output đúng schema (`review-findings.schema.json`,
  `review-report.schema.json`)
- **PHẢI** giới hạn scope của từng sub-agent (không cho agent đọc toàn repo)
- **PHẢI** ghi `routing_decisions[]` vào plan.json để audit được quyết định map

## Shortcuts

- Slash command: `/review <yêu cầu>` — xem `.claude/commands/review.md`
- Framework docs: `.claude/docs/index.html` (v0.1.2)

## Version
**Framework v0.1.2** (2026-06-14). Patches từ v0.1.1 (đến từ feedback
RV-MOD-20260614-02 — review module web):

- **Bước 10 demojibake (CRITICAL)**: `review-report-render-skill` §4.5 thêm
  pre-render pass fix double-UTF-8 mojibake (em-dash `—` thành `â€"`, curly
  quotes thành `â€™` v.v.) — fix issue persisted-output pipeline encoding,
  từng ảnh hưởng 357 chuỗi trong RV-MOD-20260614-02.
- **Bước 8 normalization**: `review-aggregation-skill` §3 thêm normalize
  pre-merge — map category aliases (`authentication`→`security`,
  `config`→`configuration`, `maintainability`→`style`) + convert
  `missing_tests` plain-string → object schema. Closes web-review-agent
  schema drift.
- **Bước 2.0 narrow `.user`**: orchestrator narrow rule — `.user` suffix CHỈ
  trigger shell prefilter khi path khớp `**/etc/udhcpc.user.d/*` (không
  apply mọi `**/*.user` như trước). Fix false-trigger với `lighttpd.user`
  (credential file).
- **Bước 7 binary-asset auto-exclude**: file extension `binary_asset_extensions`
  (png/jpg/gif/svg/woff/ttf/mp3/pdf...) giữ mapping audit trail nhưng exclude
  khỏi agent task scope. Fix web agent đọc 52 file ảnh trong
  `dl/webserver-1.0/pages/images/`.
- **Bước 7 sampling guidance**: khi domain có >20 file cùng extension, apply
  security-critical heuristic + stratified sample, hoặc multi-task partition.
  Fix 58-HTML web review không có guidance trong v0.1.1.
- **Bước 7 third-party libs**: `jquery*.js`/`*.min.js`/`bootstrap*`/`d3*` —
  out-of-scope deep review, chỉ check CVE/SRI.
- **web-review-agent** version → 0.1.2 — agent prompt thêm explicit category
  enum + missing_tests format constraint + JSON output encoding hint.
- **agent-registry.yaml** version → 0.1.2 — add `binary_asset_extensions` và
  `category_aliases` top-level keys.

v0.1.1 (2026-06-13). Patches từ v0.1.0:
- Bước 2.0 mới (shell-extension prefilter) trong orchestrator
- Tie-break rule "tail-segment specificity wins" cho domain vs domain
- `git-context-skill` fallback `git ls-tree HEAD` khi index rỗng
- `openwrt-source-context-skill` phát hiện Pattern C "pseudo-tarball in-tree"
- `tr069-review-agent` mở rộng cover `dl/vnptt-unify-1.0/{vnptt-cwmp,libbbfdm*,libcwmpdm}/**`
- `mesh-review-agent` thêm `shell-script-review-skill` + `openwrt-shell-domain-skill`
- `category` enum mở rộng 13 giá trị mới (memory-safety, concurrency, supply-chain, ...)
- Agent mới: `python-tools-review-agent` (Python build tools + Dockerfile)

v0.1.0 (stable first release, 2026-04).
