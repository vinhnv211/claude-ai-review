---
description: Multi-agent code review qua AI Review Framework v0.1.0 (8 agents, 10 bước)
argument-hint: <yêu cầu review, VD "commit gần nhất" | "3 commit" | "module pppoe" | "PR" | "commit abc123">
---

# /review — AI Multi-Agent Code Review

User yêu cầu review: **$ARGUMENTS**

## Nhiệm vụ bắt buộc

Thực hiện code review cho repo `xsw240n_firmware` qua framework multi-agent tại
`.claude/`. KHÔNG tự review bằng general knowledge. KHÔNG dùng built-in skill
`/review` hoặc `/security-review` của Claude Code.

## Quy trình

**Bước 1:** Đọc `.claude/orchestrators/review-orchestrator.md` (nếu chưa đọc turn này).

**Bước 2:** Parse `$ARGUMENTS` để xác định:
- `mode`: `diff-review` | `commit-review` | `module-review` | `full-review`
- `commit_spec` (nếu commit-review): `{ref, count, aggregation}`, validate count ∈ [1,10]
- `module_path` (nếu module-review)
- `focus_tags` (nếu user nêu: security, performance, compat…)

Mapping nhanh:
| User nói | mode | spec |
|----------|------|------|
| "commit gần nhất" / "HEAD" | commit-review | ref=HEAD, count=1 |
| "N commit gần nhất" (N≤10) | commit-review | ref=HEAD, count=N |
| "commit thứ K gần nhất" | commit-review | ref=HEAD~(K-1), count=1 |
| "commit `<hash>`" | commit-review | ref=hash, count=1 |
| "module X" / "package Y" | module-review | module_path=path/to/X |
| "PR" / "diff" / "changed files" | diff-review | — |
| "audit" / "full review" | full-review | — |

**Bước 3:** Thực thi 10 bước của orchestrator. Quan trọng:
- **0.2** Detect OpenWrt package (Makefile có `PKG_SOURCE`+`PKG_HASH`) → gọi
  `openwrt-source-context-skill` để fetch tarball, verify SHA256, apply patches,
  sinh synthetic diff + virtual path.
- **2** Map file → agent theo 3 giai đoạn:
  - 2a virtual file (có `virtual_source`) → basename trigger / extension
  - 2b path match + longest-glob tie-breaker + skip `is_fallback:true`
  - 2c fallback chain (web → openwrt-build shell → generic)
  - Ghi `routing_decisions[]` vào `plan.json`
- **6** Chỉ assign agent có id trong `.claude/schemas/agent-registry.yaml` (8 agents)
- **10** BẮT BUỘC gọi `review-report-render-skill` ghi artifacts vào
  `.review/<review_id>/` (meta/plan/report.json/md/html + findings/)

## 8 Agents hợp lệ

`dhcp-review-agent`, `pppoe-review-agent`, `tr069-review-agent`,
`mesh-review-agent`, `wifi-driver-review-agent`, `web-review-agent`,
`openwrt-build-review-agent`, `generic-review-agent`.

## Return cho user

Sau khi xong, trả về:
- `review_id`: `RV-<MODE>-<YYYYMMDD>-<SEQ>[-<shorthash>]`
- Verdict: `block` | `needs-fix` | `ship`
- Đường dẫn `report.md` + `report.html`
- Tóm tắt ngắn (stats: critical/major/minor/info + top 3 finding)

## Ví dụ invocation

```
/review commit gần nhất
/review 3 commit mới nhất focus security
/review commit abc1234
/review module package/network/services/ppp
/review PR
```
