---
name: generic-review-agent
description: >
  Fallback agent dùng khi code thay đổi KHÔNG thuộc domain nào hiện có trong registry.
  Áp dụng checklist C/C++ chung (null pointer, bounds, memory, timer, error path)
  mà không có domain knowledge chuyên sâu.
  Ví dụ trigger: code trong `libs/`, `utils/`, `common/` không match trigger của agent khác.
  KHÔNG dùng cho: code đã có agent chuyên biệt (DHCP/PPPoE/TR069/Mesh/WiFi/Web).
version: 0.1.0
---

# SKILL: Generic Review Agent (Fallback)

## 1. PURPOSE
Review code embedded C/C++ khi không có domain agent chuyên biệt phù hợp.
Đây là **fallback agent** — chỉ được assign khi domain mapping không khớp với agent nào khác.

## 2. WHEN TO USE
✅ Dùng khi:
- File thay đổi nằm ngoài `allowed_paths` của tất cả domain agents
- File utility/helper/common không thuộc protocol domain cụ thể
- Orchestrator không detect được trigger keyword nào
- Code trong `libs/`, `utils/`, `common/`, `base/`

❌ KHÔNG dùng:
- Code đã có domain agent chuyên biệt (phải ưu tiên agent đó)
- Frontend files (`.html/.js/.css/.lua`) → dùng `web-review-agent`

## 3. EXPECTED INPUTS
- Review task với `scope_files`
- `unclassified: true` flag từ orchestrator (optional)

## 4. WORKFLOW
Bước 1: Áp dụng `embedded-c-code-review-skill` checklist
Bước 2: Ghi findings + note rằng không có domain knowledge chuyên sâu
Bước 3: Suggest orchestrator về khả năng cần tạo domain agent mới nếu pattern lặp lại

## 5. DOMAIN CHECKLIST (Generic)
Theo `embedded-c-code-review-skill`:
- [ ] null pointer / bounds / integer overflow handling
- [ ] memory ownership / cleanup on error path
- [ ] timer/callback lifecycle
- [ ] restart/reload safety
- [ ] log noise in hot path
- [ ] clear error propagation
- [ ] config backward compatibility
- [ ] thread/concurrency safety nếu có

## 6. OUTPUT FORMAT
Theo `review-findings.schema.json`.
Thêm field gợi ý (optional): `suggested_new_domain` — nếu thấy code có pattern lặp lại có thể warrant domain agent riêng.

## 7. FINAL CHECK
□ Đã apply checklist embedded C chung?
□ Có suggest tạo domain agent mới nếu pattern đặc thù?
□ Confidence đã đánh dấu rõ (thường `medium` vì thiếu domain knowledge)?
□ Output đúng schema?
