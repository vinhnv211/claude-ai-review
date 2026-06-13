---
name: review-report-render-skill
description: >
  Skill này nên được dùng khi cần render aggregated review report từ JSON sang
  các định dạng dễ đọc cho người (Markdown, HTML) và chuẩn hóa thư mục/đặt tên
  file output. Dùng bởi review-orchestrator ở Bước 10 (Final Output).
  KHÔNG dùng cho: tạo finding mới, phân tích code, merge findings (đã làm ở
  review-aggregation-skill).
version: 0.1.2
---

# SKILL: Review Report Render

<!--
Changelog:
  0.1.2 (2026-06-14):
    - Add §4.5 "Demojibake post-process": detect và fix double-UTF-8-encoded
      sequences (em-dash, curly quotes, NBSP) bị mojibake bởi
      persisted-output pipeline. PHẢI chạy trước khi ghi artifacts.
      Closes RV-MOD-20260614-02 feedback issue #1.
-->


## 1. PURPOSE
Chuyển `review-report.schema.json` → các artifact output người đọc được (Markdown,
HTML) và đảm bảo mọi review chạy ra cùng một cấu trúc thư mục + naming convention.
Output MD cũng là đầu vào cho automation (CI comment, dashboard, diff bot).

## 2. WHEN TO USE
✅ Dùng sau khi aggregation hoàn tất và có `review-report.schema.json` hợp lệ.
✅ Dùng để chuẩn hóa thư mục output + đặt tên file.
❌ KHÔNG dùng để tạo/sửa findings — chỉ format lại.

## 3. OUTPUT DIRECTORY CONVENTION

### 3.1 Root
Mặc định: `.review/` ở repo root. Config qua env `REVIEW_OUTPUT_ROOT`.

### 3.2 Review ID
Format: `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`

| Mode | Prefix | Có shorthash? |
|------|--------|---------------|
| diff-review     | `DIFF` | không |
| commit-review   | `CMT`  | có — short hash 7 ký tự của commit mới nhất trong range |
| full-review     | `FULL` | không |
| module-review   | `MOD`  | không |

`SEQ` = 2-digit counter trong ngày, bắt đầu từ `01`.

Ví dụ:
- `RV-CMT-20260418-01-abc123d`  — review commit gần nhất
- `RV-CMT-20260418-02-f9e88a1`  — review 3 commit gần nhất (hash mới nhất)
- `RV-MOD-20260418-01`          — review module dhcp
- `RV-DIFF-20260418-01`         — review PR patch
- `RV-FULL-20260418-01`         — review một file cụ thể

### 3.3 Thư mục 1 review
```
.review/
└── RV-CMT-20260418-01-abc123d/
    ├── meta.json          # metadata ngắn: id, mode, generated_at, verdict, stats
    ├── plan.json          # review plan (theo review-plan.schema.json)
    ├── report.json        # aggregated report (theo review-report.schema.json)
    ├── report.md          # render Markdown cho người đọc
    ├── report.html        # render HTML standalone (self-contained CSS)
    └── findings/
        ├── dhcp-review-agent.json
        ├── web-review-agent.json
        └── generic-review-agent.json
```

### 3.4 Index tổng
`.review/index.json` — list toàn bộ reviews đã chạy, sort theo `generated_at` desc.
Mỗi entry: `{review_id, mode, generated_at, verdict, stats.total, path}`.

## 4.5 DEMOJIBAKE POST-PROCESS (CRITICAL v0.1.2)

Persisted-output pipeline (khi sub-agent return > 50KB) có nguy cơ
**double-encode UTF-8** thành CP1252 rồi re-encode UTF-8, biến `—` thành
chuỗi `â€"` v.v. Mojibake này ẩn trong JSON string nên schema validation
KHÔNG bắt được. Reviewer đọc report sẽ thấy text bị méo.

PHẢI chạy demojibake pass cho **mọi text field** trong report TRƯỚC khi ghi
`report.md`/`.html`/`.json`. Apply cho:
- `summary`, `verdict_rationale`
- `findings[*].title`, `evidence`, `code_snippet`, `impact`, `recommendation`
- `risk_summary.*`, `human_attention[*].reason`
- `missing_tests[*].description`

### 4.5.1 Mojibake → Unicode mapping table

```python
MOJI_MAP = {
    # Em-dash / en-dash
    "â€”": "—",   # U+2014 EM DASH
    "â€“": "–",   # U+2013 EN DASH  (less common, but seen)
    "â€\"": "—",       # variant — observed in RV-MOD-20260614-02
    "â€“": "–",        # en-dash variant

    # Curly quotes
    "â€™": "'",        # U+2019 RIGHT SINGLE QUOTATION MARK
    "â€˜": "'",        # U+2018 LEFT SINGLE QUOTATION MARK
    "â€œ": '"',        # U+201C LEFT DOUBLE QUOTATION MARK
    "â€": '"',   # U+201D RIGHT DOUBLE (raw form)
    "â€â€¦": "..", # weird drift
    "â€¦": "...",      # U+2026 HORIZONTAL ELLIPSIS

    # Non-breaking space (and its mojibake form)
    "Â\xa0": " ",
    "Â ": " ",         # narrow form

    # Other common Vietnamese diacritic mojibake
    "Ã©": "é", "Ã¨": "è", "Ã ": "à", "Ã²": "ò", "Ã³": "ó",
    "Ã¹": "ù", "Ã¼": "ü", "Ã¶": "ö", "Ã±": "ñ",
}
```

### 4.5.2 Algorithm

```python
def demojibake(text: str) -> str:
    if not text:
        return text
    out = text
    # Apply specific mappings (longest match first to avoid greedy collapse)
    for bad, good in sorted(MOJI_MAP.items(), key=lambda kv: -len(kv[0])):
        out = out.replace(bad, good)
    return out

def normalize_report(report: dict) -> dict:
    # Walk all string fields recursively
    def walk(node):
        if isinstance(node, str):
            return demojibake(node)
        if isinstance(node, list):
            return [walk(x) for x in node]
        if isinstance(node, dict):
            return {k: walk(v) for k, v in node.items()}
        return node
    return walk(report)
```

### 4.5.3 Verification gate

Sau demojibake, scan output cho residual:
```python
RESIDUAL_PATTERNS = ["â€", "Ã©", "Ã ", "Ã¨", "Â\xa0"]
def check_residual(text: str) -> list[str]:
    return [p for p in RESIDUAL_PATTERNS if p in text]
```

Nếu còn residual sau demojibake → log `routing_warning` với
`"residual_mojibake: <pattern>"` và **không block** render (best-effort cleanup).
Mojibake không phải security bug, chỉ là display issue.

### 4.5.4 Apply trước khi ghi

Workflow:
1. Aggregator emit `report.json` (raw, có thể chứa mojibake)
2. Render skill **load** report.json → in-memory dict
3. **Apply `normalize_report(R)` ngay sau load**
4. Compute verdict + top_findings + render MD/HTML từ dict đã normalize
5. **Re-write `report.json` với version đã clean** (để downstream automation cũng đọc bản sạch)
6. Ghi `report.md`, `report.html` từ dict đã clean

## 4. NAMING RULES

| Artifact | Tên cố định |
|----------|-------------|
| Aggregated report | `report.json`, `report.md`, `report.html` |
| Per-agent findings | `findings/<agent-id>.json` (trùng id trong registry) |
| Orchestration plan | `plan.json` |
| Metadata ngắn | `meta.json` |
| Index tổng | `.review/index.json` |

Không đặt tên có dấu, space, hoặc timestamp trong tên file — timestamp đã ở `review_id`.

## 5. MARKDOWN TEMPLATE

Render `review-report.schema.json` theo layout dưới đây. Giữ thứ tự section cố định
để downstream parser (CI bot, dashboard) có thể grep ổn định.

```markdown
# Review Report `{{review_id}}`

**Verdict:** {{verdict_badge}} `{{verdict}}` — {{verdict_rationale}}
**Mode:** `{{mode}}` · **Generated:** {{generated_at}}
**Stats:** 🔴 {{stats.critical}} critical · 🟠 {{stats.major}} major · 🟡 {{stats.minor}} minor · ⚪ {{stats.info}} info · **Total {{stats.total}}**

## Scope
{{#if commit_spec}}
- **Commits ({{commit_spec.count}}):**
{{#each commits}}
  - `{{hash_short}}` {{subject}} — _{{author}}, {{date}}_
{{/each}}
{{/if}}
{{#if module_path}}
- **Module:** `{{module_path}}`
{{/if}}
- **Files reviewed ({{scope.files.length}}):** {{first 5 files, ellipsize}}
{{#if focus_tags}}
- **Focus:** {{focus_tags joined}}
{{/if}}

## TL;DR
{{summary}}

## Top Findings
{{#each top_findings as fid}}
{{render_finding_block(findings[fid])}}
{{/each}}

## Findings by Severity

### 🔴 Critical ({{count}})
<details><summary>Expand</summary>

{{#each critical_findings}}
{{render_finding_block}}
{{/each}}
</details>

### 🟠 Major ({{count}})
<details><summary>Expand</summary>
... (same pattern)
</details>

### 🟡 Minor ({{count}})
<details>...</details>

### ⚪ Info ({{count}})
<details>...</details>

## Findings by Domain
| Domain | Agent | Count | Critical | Major |
|--------|-------|------:|---------:|------:|
{{#each agents_run}}
| {{domain}} | `{{agent}}` | {{finding_count}} | {{critical}} | {{major}} |
{{/each}}

## Risk Summary
- **Regression:** {{risk_summary.regression}}
- **Compatibility:** {{risk_summary.compatibility}}
- **Performance:** {{risk_summary.performance}}
- **Security:** {{risk_summary.security}}

## Missing Tests
{{#each missing_tests}}
- [ ] **[{{id}}]** {{description}} _(priority: {{priority}}; covers: {{related_findings joined}})_
{{/each}}

## Human Attention
{{#each human_attention}}
- **{{area}}** — {{reason}} (related: {{related_findings joined}})
{{/each}}

## Suggested New Agents
{{#each suggested_new_agents}}
- **`{{suggested_domain}}`** — {{rationale}}
  - Sample: {{sample_files joined}}
{{/each}}

---
_Generated by review-orchestrator v{{framework_version}} — artifacts: `plan.json`, `report.json`, `findings/`_
```

### 5.1 Finding block template
Mỗi finding render thành khối sau:
```markdown
#### {{severity_icon}} [{{id}}] {{title}}
- **File:** `{{file}}{{#if line}}:{{line}}{{#if line_end}}-{{line_end}}{{/if}}{{/if}}`{{#if function}} · `{{function}}()`{{/if}}
- **Agent:** `{{agent}}`{{#if sub_skill}} / `{{sub_skill}}`{{/if}} · **Category:** {{category}} · **Confidence:** {{confidence}}
- **Evidence:**
  ```{{lang}}
  {{code_snippet or evidence}}
  ```
- **Impact:** {{impact}}
- **Fix:** {{recommendation}}
{{#if references}}
- **Refs:** {{references joined}}
{{/if}}
```

Icon mapping:
- `critical` → 🔴
- `major` → 🟠
- `minor` → 🟡
- `info` → ⚪
- Verdict: `block` → 🔴, `needs-fix` → 🟠, `ship` → 🟢

## 6. HTML TEMPLATE

HTML standalone (no external assets), copy-paste-friendly. Sử dụng CSS nhỏ gọn
+ `<details>` để collapse section dài. Layout:

```html
<!DOCTYPE html>
<html lang="vi"><head>
<meta charset="utf-8">
<title>Review Report {{review_id}}</title>
<style>
  :root { --crit:#c0392b; --maj:#e67e22; --min:#f1c40f; --info:#95a5a6;
          --bg:#0f1419; --panel:#1a2028; --fg:#d7dce3; --muted:#8a93a0; --acc:#4aa3df; }
  * { box-sizing: border-box; }
  body { margin:0; font-family:-apple-system,Segoe UI,Roboto,sans-serif;
         background:var(--bg); color:var(--fg); line-height:1.5; }
  .wrap { max-width: 1100px; margin: 0 auto; padding: 24px; }
  h1 { font-size:1.6rem; margin:0 0 8px; }
  h2 { border-bottom:1px solid #2a323d; padding-bottom:6px; margin-top:32px; }
  .verdict { display:inline-block; padding:4px 10px; border-radius:6px;
             font-weight:600; font-size:.9rem; }
  .verdict.block { background:var(--crit); color:#fff; }
  .verdict.needs-fix { background:var(--maj); color:#fff; }
  .verdict.ship { background:#27ae60; color:#fff; }
  .stats { display:flex; gap:12px; flex-wrap:wrap; margin:12px 0; }
  .stat { background:var(--panel); padding:8px 14px; border-radius:6px;
          border-left:4px solid var(--muted); min-width:90px; }
  .stat.crit { border-color:var(--crit); }
  .stat.maj  { border-color:var(--maj); }
  .stat.min  { border-color:var(--min); }
  .stat .n { font-size:1.4rem; font-weight:700; display:block; }
  .finding { background:var(--panel); border-radius:8px; padding:14px 18px;
             margin:12px 0; border-left:4px solid var(--muted); }
  .finding.critical { border-color:var(--crit); }
  .finding.major    { border-color:var(--maj); }
  .finding.minor    { border-color:var(--min); }
  .finding h4 { margin:0 0 6px; font-size:1.05rem; }
  .finding .meta { color:var(--muted); font-size:.85rem; margin:4px 0 10px; }
  .finding pre { background:#0a0d12; padding:10px 12px; border-radius:6px;
                 overflow-x:auto; font-size:.85rem; margin:8px 0; }
  .finding .label { color:var(--acc); font-weight:600; }
  table { border-collapse: collapse; width:100%; margin:12px 0; }
  th, td { border:1px solid #2a323d; padding:6px 10px; text-align:left; }
  th { background:var(--panel); }
  details { background:var(--panel); border-radius:6px; padding:10px 14px; margin:10px 0; }
  summary { cursor:pointer; font-weight:600; }
  code { background:#0a0d12; padding:1px 6px; border-radius:3px; font-size:.9em; }
  .tldr { background:var(--panel); border-left:4px solid var(--acc);
          padding:12px 16px; border-radius:4px; }
  footer { margin-top:40px; padding-top:16px; border-top:1px solid #2a323d;
           color:var(--muted); font-size:.85rem; }
</style>
</head><body><div class="wrap">
  <h1>Review Report <code>{{review_id}}</code></h1>
  <p>
    <span class="verdict {{verdict}}">{{verdict_upper}}</span>
    &nbsp; mode: <code>{{mode}}</code>
    &nbsp; generated: {{generated_at}}
  </p>
  <div class="stats">
    <div class="stat crit"><span class="n">{{stats.critical}}</span>critical</div>
    <div class="stat maj"><span class="n">{{stats.major}}</span>major</div>
    <div class="stat min"><span class="n">{{stats.minor}}</span>minor</div>
    <div class="stat"><span class="n">{{stats.info}}</span>info</div>
    <div class="stat"><span class="n">{{stats.total}}</span>total</div>
  </div>

  <h2>Scope</h2>
  <!-- render commits / module / files -->

  <h2>TL;DR</h2>
  <div class="tldr">{{summary}}</div>

  <h2>Top Findings</h2>
  <!-- each finding as .finding.{severity} block -->

  <h2>Findings by Severity</h2>
  <details open><summary>🔴 Critical ({{n}})</summary>...</details>
  <details><summary>🟠 Major ({{n}})</summary>...</details>
  <details><summary>🟡 Minor ({{n}})</summary>...</details>
  <details><summary>⚪ Info ({{n}})</summary>...</details>

  <h2>Findings by Domain</h2>
  <table>...</table>

  <h2>Risk Summary</h2>
  <ul>...</ul>

  <h2>Missing Tests</h2>
  <ul>...</ul>

  <h2>Human Attention</h2>
  <ul>...</ul>

  <h2>Suggested New Agents</h2>
  <ul>...</ul>

  <footer>
    Generated by review-orchestrator v{{framework_version}} ·
    Artifacts: <code>plan.json</code>, <code>report.json</code>, <code>findings/</code>
  </footer>
</div></body></html>
```

### Finding HTML block
```html
<div class="finding {{severity}}">
  <h4>[{{id}}] {{title}}</h4>
  <div class="meta">
    <code>{{file}}:{{line}}</code> · agent <code>{{agent}}</code>
    · {{category}} · confidence: {{confidence}}
  </div>
  <pre><code>{{code_snippet or evidence}}</code></pre>
  <p><span class="label">Impact:</span> {{impact}}</p>
  <p><span class="label">Fix:</span> {{recommendation}}</p>
  {{#if references}}<p><span class="label">Refs:</span> {{references joined}}</p>{{/if}}
</div>
```

## 7. VERDICT LOGIC

Auto-compute verdict nếu không có sẵn trong report:

| Condition | Verdict |
|-----------|---------|
| Có ≥1 finding `critical` (bất kỳ confidence nào) | `block` |
| Có ≥1 finding `major` với confidence `high` | `block` |
| Có finding `major` hoặc `minor` (nhưng không rơi vào block) | `needs-fix` |
| Chỉ có `info` hoặc zero findings | `ship` |

`verdict_rationale`: 1 câu ngắn, ví dụ:
- `"2 critical (DHCP-001, DHCP-004) + 1 major confidence=high (WEB-CGI-002)"`
- `"3 major, 5 minor — không có critical"`
- `"sạch, 2 info"`

## 8. TOP FINDINGS RANKING

Rank findings theo score giảm dần:
```
severity_weight: critical=100, major=40, minor=10, info=1
confidence_weight: high=1.0, medium=0.7, low=0.4
category_weight: security=1.3, correctness=1.1, reliability=1.0, compat=0.9, performance=0.8, style=0.5

score = severity_weight × confidence_weight × category_weight
```
Lấy top 5-10 (tối đa 10), chỉ lưu `id` trong `top_findings` — findings đầy đủ nằm trong `findings[]`.

## 9. WORKFLOW

**Bước 1:** Nhận `review-report.schema.json` từ orchestrator
**Bước 1.5 (NEW v0.1.2):** **Apply `demojibake` pass cho toàn bộ report dict (§4.5)** — TRƯỚC mọi tính toán, để verdict_rationale/summary clean.
**Bước 2:** Tính `verdict` + `verdict_rationale` nếu chưa có
**Bước 3:** Tính `top_findings` theo ranking
**Bước 4:** Xác định `review_id` + tạo thư mục `.review/<review_id>/`
**Bước 5:** Ghi artifacts theo thứ tự:
  1. `meta.json` (ngắn, để index đọc nhanh)
  2. `plan.json`
  3. `findings/<agent>.json` (copy từ per-agent output — cũng demojibake trước khi ghi)
  4. `report.json` (aggregated, đã demojibake)
  5. `report.md` (render từ template section 5)
  6. `report.html` (render từ template section 6)
**Bước 6:** Update `.review/index.json` — prepend entry mới
**Bước 7:** Scan tất cả artifacts vừa ghi với `check_residual` (§4.5.3). Nếu residual > 0 → log vào meta.json `residual_mojibake_count`.
**Bước 8:** Return đường dẫn `report.md` + `report.html` + `review_id`

## 10. META.JSON FORMAT
```json
{
  "review_id": "RV-CMT-20260418-01-abc123d",
  "mode": "commit-review",
  "generated_at": "2026-04-18T09:15:22Z",
  "verdict": "block",
  "stats": {"critical": 2, "major": 5, "minor": 8, "info": 3, "total": 18},
  "scope_summary": "3 commits on HEAD, 12 files",
  "artifacts": {
    "plan": "plan.json",
    "report_json": "report.json",
    "report_md": "report.md",
    "report_html": "report.html",
    "raw_findings_dir": "findings/"
  }
}
```

## 11. GUARDRAILS
- KHÔNG thêm/sửa nội dung finding trong khi render (TRỪ demojibake — đó là sửa encoding, không phải nội dung)
- KHÔNG đổi id của finding
- PHẢI giữ thứ tự section cố định trong MD (parser-friendly)
- PHẢI escape HTML khi render `evidence`/`code_snippet` vào HTML
- PHẢI đảm bảo HTML self-contained (inline CSS, không link external)
- PHẢI ghi atomically: ghi vào `.tmp` rồi rename, tránh corrupt khi crash
- PHẢI tạo thư mục cha nếu chưa tồn tại
- PHẢI apply demojibake pass (§4.5) trước khi ghi BẤT KỲ artifact nào

## 12. DEPENDENCIES
- `schemas/review-report.schema.json`
- `schemas/review-findings.schema.json`
- Input từ `review-aggregation-skill`

## 13. FINAL CHECK
□ review_id đúng format `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<hash>]`?
□ Đã tạo đủ 4 artifact chính (report.json/md/html, plan.json)?
□ Per-agent findings đã copy vào `findings/`?
□ meta.json + index.json đã update?
□ Verdict + top_findings đã compute?
□ HTML self-contained (không external asset)?
□ MD giữ đúng thứ tự section?
□ **Demojibake pass đã chạy cho TẤT CẢ artifacts (§4.5)?**
□ **`check_residual` scan đã chạy + log vào meta.json nếu có residual?**
