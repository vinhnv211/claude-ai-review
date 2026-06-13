---
name: review-orchestrator
description: >
  Dùng skill này khi user yêu cầu code review dưới bất kỳ dạng nào:
  "review PR", "review code", "review firmware",
  "review commit gần nhất", "review HEAD",
  "review N commit gần nhất" (N ≤ 10, ví dụ "review 3 commit mới nhất"),
  "review commit thứ K gần nhất" (K ≤ 10, ví dụ "review commit thứ 2 gần nhất"),
  "review commit <hash>" (commit ID cụ thể),
  "review module X", "review toàn bộ package Y",
  "analyze git diff", "impact analysis before merge",
  "check code quality", "audit code", "kiểm tra code", "đánh giá code".
  Skill này điều phối multi-agent code review cho dự án embedded / OpenWrt:
  Bước 0 (git context + OpenWrt tarball detection) → Bước 1-9 (map domain,
  assign agents, aggregate findings) → Bước 10 (render Markdown + HTML +
  ghi artifacts vào .review/<review_id>/).
  KHÔNG dùng cho: review từng dòng code chi tiết, debug root cause sâu,
  viết patch fix, thiết kế kiến trúc mới, built-in GitHub PR review.
version: 0.1.0
---

# SKILL: Review Orchestrator (entry wrapper)

## PURPOSE
Đây là entry wrapper để Claude Code auto-trigger framework AI Review Multi-Agent
khi user gửi prompt match description trên. Toàn bộ logic chi tiết nằm ở file
canonical `.claude/orchestrators/review-orchestrator.md` — skill này chỉ route tới
đó và liệt kê guardrail tối quan trọng.

## BẮT BUỘC ngay khi skill được invoke

**Bước 1:** Đọc toàn bộ file canonical orchestrator:
```
.claude/orchestrators/review-orchestrator.md
```
File này định nghĩa workflow 10 bước chi tiết, schema I/O, rule routing
tie-breaker, và final check list. KHÔNG skip.

**Bước 2:** Đọc registry để biết 8 agent hợp lệ:
```
.claude/schemas/agent-registry.yaml
```
Không assign agent ngoài registry.

**Bước 3:** Thực thi theo đúng 10 bước trong file canonical. Tóm tắt nhanh:

| Bước | Nội dung | Skill/File dùng |
|------|----------|-----------------|
| 0.1 | Git context (resolve commit_spec / module ls-files) | `skills/core/git-context-skill.md` |
| 0.2 | OpenWrt tarball detect + fetch + verify + patch + synthetic diff | `skills/core/openwrt-source-context-skill.md` |
| 1 | Change Understanding | orchestrator §5 Bước 1 |
| 2 | Domain Mapping (2a virtual → 2b longest-glob → 2c fallback) | `skills/core/review-domain-mapping-skill.md` |
| 3 | Unclassified Handling (gợi ý agent mới) | orchestrator §5 Bước 3 |
| 4-5 | Strategy + Task Decomposition | orchestrator §5 |
| 6 | Agent Selection (từ registry) | `schemas/agent-registry.yaml` |
| 7 | Scope Control (scope tối thiểu mỗi agent) | orchestrator §5 |
| 8 | Aggregation (merge/dedupe) | `skills/core/review-aggregation-skill.md` |
| 9 | Gap Detection (test / compat / performance) | `skills/core/review-risk-skill.md` |
| 10 | Final Output (verdict + render MD/HTML + ghi artifacts) | `skills/core/review-report-render-skill.md` |

## GUARDRAILS tối quan trọng (không bao giờ vi phạm)

- **KHÔNG** dùng built-in skill `/review` hoặc `/security-review` (GitHub PR) —
  đây là skill riêng của repo này.
- **KHÔNG** tự review code bằng general knowledge — phải đi qua multi-agent pipeline.
- **KHÔNG** assign agent ngoài `.claude/schemas/agent-registry.yaml`.
- **KHÔNG** skip Bước 0.2 nếu repo có `package/**/Makefile` (OpenWrt pattern).
- **KHÔNG** skip Bước 10. Không có `.review/<review_id>/report.md` = review
  chưa hoàn thành.
- **PHẢI** validate `commit_spec.count ∈ [1, 10]` — > 10 → reject, yêu cầu user giảm.
- **PHẢI** ghi `routing_decisions[]` vào `plan.json` để audit được quyết định map.

## OUTPUT expected
Sau khi hoàn thành, return cho user:
1. `review_id` theo format `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`
2. Đường dẫn `.review/<review_id>/report.md` (cho người đọc)
3. Đường dẫn `.review/<review_id>/report.html` (self-contained)
4. Verdict: `block` | `needs-fix` | `ship`

## 8 Agents hợp lệ (v0.1.0)
`dhcp-review-agent` · `pppoe-review-agent` · `tr069-review-agent` ·
`mesh-review-agent` · `wifi-driver-review-agent` · `web-review-agent` ·
`openwrt-build-review-agent` · `generic-review-agent`

## Tham chiếu
- Orchestrator canonical: `.claude/orchestrators/review-orchestrator.md`
- Extension guide: `.claude/orchestrators/review-extension-guide.md`
- Docs: `.claude/docs/index.html`
- Registry: `.claude/schemas/agent-registry.yaml`
