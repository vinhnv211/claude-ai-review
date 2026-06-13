---
name: review-risk-skill
description: >
  Skill này nên được dùng khi cần đánh giá risk từ code change,
  ví dụ: "regression risk", "compatibility risk", "performance risk", "state machine risk",
  "security risk", "lifecycle risk". Output là risk summary.
  KHÔNG dùng cho: benchmark hoặc profiling chi tiết.
version: 0.1.0
---

# SKILL: Review Risk

## 1. PURPOSE
Đánh giá risk không chỉ ở bug hiện tại mà còn ở regression, compatibility, lifecycle,
performance và security. Bổ trợ cho findings từ agents bằng cái nhìn cross-cutting.

## 2. RISK CATEGORIES

### 2.1 Regression
- Thay đổi behavior của path đã tested
- Side effect sang module khác qua shared state
- Protocol compliance bị phá vỡ (timing, sequencing)

### 2.2 Backward compatibility
- Config schema thay đổi (UCI option bị đổi tên/xóa)
- Parameter TR-069/CWMP bị đổi type
- API endpoint frontend bị đổi contract
- Default value thay đổi ảnh hưởng upgrade path

### 2.3 Restart / Reload lifecycle
- Daemon restart mất state không khôi phục được
- `uci commit` + service reload không đồng bộ
- Init sequence bị xáo trộn

### 2.4 Memory / Resource
- Memory leak path mới (error path chưa free)
- File descriptor leak
- Timer/callback không cancel khi cleanup
- Kernel memory trong driver (GFP flags sai)

### 2.5 Performance / Latency
- Thêm syscall trong hot path
- Log verbose trong interrupt/packet path
- Lock contention mới
- O(n²) trong loop xử lý packet/event

### 2.6 Security exposure
- **Web/CGI**: command injection, XSS, CSRF, auth bypass, path traversal
- **Protocol**: malformed packet handling (DHCP option, PPP LCP, CWMP SOAP)
- **Config**: hardcoded credentials, weak defaults
- **Driver**: OOB read/write trong vendor hook

## 3. RISK ASSESSMENT WORKFLOW

Cho mỗi change:
1. Xét 6 categories → rate `high / medium / low / none`
2. Nếu rate ≥ medium → cần mitigation plan hoặc additional test
3. Tổng hợp thành `risk_summary`

## 4. OUTPUT
```json
{
  "risk_summary": {
    "regression":           {"level": "medium", "note": "..."},
    "backward_compatibility": {"level": "low", "note": "..."},
    "lifecycle":            {"level": "high", "note": "service reload sau UCI commit chưa rõ"},
    "memory_resource":      {"level": "low"},
    "performance":          {"level": "none"},
    "security":             {"level": "high", "note": "CGI handler thiếu auth check"}
  },
  "mitigation_needed": [
    "Add test: WAN flap during active PPPoE session",
    "Add auth check in CGI handler apply.cgi"
  ]
}
```

## 5. CROSS-REFERENCES
- Security risk thường đã được các sub-skill detect ở level finding
  (xem `web-cgi-skill`, `web-javascript-skill`, `dhcp-domain-skill`)
- Risk skill NÂNG lên level orchestrator: tổng hợp + ưu tiên
- Risk level cao → orchestrator set priority cho task tương ứng
