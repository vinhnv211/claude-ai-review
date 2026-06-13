---
name: review-aggregation-skill
description: >
  Skill này nên được dùng khi cần tổng hợp findings từ nhiều sub-agent,
  ví dụ: "merge review results", "aggregate findings", "deduplicate issues",
  "combine per-commit findings". Output là report review thống nhất.
  KHÔNG dùng cho: review code gốc.
version: 0.1.2
---

# SKILL: Review Aggregation

<!--
Changelog:
  0.1.2 (2026-06-14):
    - Add §3 "Normalization pre-merge": agent output schema lệch (category
      aliases, missing_tests as plain strings, missing fields) phải được
      normalize TRƯỚC khi merge. Map alias theo agent-registry.yaml
      category_aliases; convert plain-string missing_tests → object.
      Closes RV-MOD-20260614-02 feedback issues #2, #4.
    - Add §2.6: image/binary asset findings auto-flagged as routing_warning
      if scope reaches agent (defense in depth — orchestrator Bước 7 should
      catch first).
-->

## 1. PURPOSE
Tổng hợp findings từ nhiều sub-agent thành report thống nhất, không trùng lặp,
có ưu tiên rõ. Hỗ trợ các nguồn findings khác nhau: domain agents, generic fallback,
per-commit reviews.

## 2. RULES

### 2.1 Dedupe
- Dedupe issue giống nhau theo tuple `(file, function, title, impact)`
- Nếu cùng issue xuất hiện ở nhiều agent → giữ severity cao nhất
- Merge `evidence` và `recommendation` nếu bổ sung cho nhau (không duplicate text)

### 2.2 Grouping
Group theo thứ tự ưu tiên:
1. **Severity** (critical → major → minor → info)
2. **Domain** (trong cùng severity)
3. **File** (trong cùng domain)

### 2.3 Phân loại action
Tách thành 3 nhóm cho consumer (reviewer/pipeline):
- **must-fix**: severity = critical hoặc (major + confidence ≥ medium)
- **should-fix**: severity = major (confidence low) hoặc minor
- **observation**: severity = info

### 2.4 Per-commit aggregation
Khi `aggregation: "per-commit"`:
- Giữ findings riêng theo commit
- Cho phép query theo commit hash
- Nếu cùng finding xuất hiện ở nhiều commit → note "first seen in commit X, still present in Y"

### 2.5 Generic + Domain overlap
Khi `generic-review-agent` và domain agent cùng review (edge case):
- Ưu tiên domain agent findings (có chuyên môn)
- Giữ generic findings nếu chúng nêu issue generic mà domain agent không cover

### 2.6 Binary asset findings (v0.1.2)
Nếu một agent trả về finding có `file` khớp `binary_asset_extensions`
(xem `agent-registry.yaml`), log `routing_warnings[]` với reason
`"binary asset reached agent scope — orchestrator Bước 7 missed exclusion"`.
Vẫn giữ finding nhưng cảnh báo về scope-control bug.

## 3. NORMALIZATION PRE-MERGE (NEW v0.1.2)

Trước khi merge findings từ các agent, aggregator PHẢI normalize output để
đảm bảo conform với `review-findings.schema.json`. Đây là defense-in-depth
cho trường hợp agent prompt-driven enforcement không đủ.

### 3.1 Category enum normalization

Step 1: load `category_aliases` từ `agent-registry.yaml`.

Step 2: với mỗi finding, áp dụng pipeline:
```
raw_cat = finding.category
if raw_cat in VALID_ENUM:
    final_cat = raw_cat              # OK
elif raw_cat in category_aliases:
    final_cat = category_aliases[raw_cat]
    log routing_warning(finding.id, f"category alias '{raw_cat}' → '{final_cat}'")
else:
    final_cat = "correctness"        # safe default
    log routing_warning(finding.id, f"unknown category '{raw_cat}' → 'correctness'")
finding.category = final_cat
```

`VALID_ENUM` (source: `review-findings.schema.json#/properties/findings/items/properties/category/enum`):
```
security, correctness, reliability, performance, compat, style, test-gap,
memory-safety, concurrency, state-machine, crypto, protocol, configuration,
supply-chain, data-model, init-script, socket-lifecycle, persistence,
scalability, architecture
```

### 3.2 `missing_tests` format normalization

Một số agent (đặc biệt web-review-agent CGI task) trả `missing_tests` dạng
plain string array thay vì object array. Aggregator PHẢI convert:

```python
def normalize_missing_test(mt, idx, agent_id):
    if isinstance(mt, str):
        return {
            "id": f"MT-{agent_id.upper()}-{idx:02d}",
            "description": mt,
            "related_findings": [],
            "priority": "medium",  # safe default
        }
    elif isinstance(mt, dict):
        # ensure required fields
        return {
            "id": mt.get("id", f"MT-{agent_id.upper()}-{idx:02d}"),
            "description": mt.get("description", mt.get("area", "?")),
            "related_findings": mt.get("related_findings", []),
            "priority": mt.get("priority", "medium"),
        }
```

### 3.3 Required field defaults

Nếu finding thiếu `confidence` → default `"medium"` + log warning.
Nếu thiếu `category` → "correctness" + log warning.
Nếu thiếu `evidence`/`impact`/`recommendation` → KHÔNG default (reject finding,
log error vì đó là core fields).

### 3.4 `missing_context` lệch level

Một số agent emit `missing_context[]` ở agent-output level. Đó là field
thuộc plan.json (orchestrator level). Aggregator PHẢI:
- Move nội dung sang `report.json.missing_context` (top-level, append)
- Log warning vào routing_warnings

## 4. OUTPUT STRUCTURE
```json
{
  "executive_summary": "...",
  "counts_by_severity": {"critical": 2, "major": 5, "minor": 3, "info": 1},
  "counts_by_domain": {"pppoe": 4, "web": 3, "generic": 1},
  "must_fix": [...],
  "should_fix": [...],
  "observation": [...],
  "findings_by_domain": {
    "pppoe": [...],
    "web": [...]
  },
  "missing_tests_aggregated": [...],
  "routing_warnings": [
    {"finding_id": "WEB-UI-005", "reason": "category alias 'config' → 'configuration'"},
    {"finding_id": "WEB-CGI-001", "reason": "missing_tests was plain string array, normalized"}
  ],
  "suggested_new_agents": [...]
}
```

## 5. QUALITY CHECKS
- [ ] Không có 2 findings giống hệt nhau (after dedupe)
- [ ] Severity counts khớp với sum per-domain
- [ ] Mọi finding có đủ `evidence`/`impact`/`recommendation`/`confidence`
- [ ] **Mọi finding có `category` thuộc enum hợp lệ** (sau normalization)
- [ ] **Mọi `missing_tests` entry là object có `description` field**
- [ ] `routing_warnings[]` được merge từ orchestrator-level và agent-level
- [ ] `suggested_new_agents` được merge từ orchestrator-level và agent-level suggestions
