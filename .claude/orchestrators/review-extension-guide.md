# Review Framework Extension Guide
version: 0.1.0

## Mục tiêu
Mở rộng framework bằng cách thêm sub-agent hoặc skill mới mà **KHÔNG** phá cấu trúc gốc.
Tất cả mở rộng đi qua `agent-registry.yaml` + file markdown theo template chuẩn.
Orchestrator chỉ đọc registry + schema — domain mới fit vào contract là hoạt động ngay.

---

## PHẦN 1: THÊM AGENT MỚI

### Khi nào cần thêm agent?
- Có một domain kỹ thuật rõ ràng chưa được cover (firewall, NAT, camera RTSP, system init)
- Domain mới có bug pattern riêng biệt, không phù hợp với agent hiện có
- Scope file path rõ ràng, không overlap với agent khác
- `generic-review-agent` liên tục báo cùng pattern → orchestrator đã emit `suggested_new_agents`

### Bước 1: Thêm registry entry
Cập nhật `schemas/agent-registry.yaml`:
```yaml
- id: <domain>-review-agent
  type: review-agent
  domain: <domain>
  capabilities: [cap1, cap2, ...]
  triggers: [keyword1, keyword2, ...]
  allowed_paths:
    - package/**/<domain>/**
    - vendor/**/<domain>/**
    - src/**/<domain>/**
  required_skills:
    - embedded-c-code-review-skill
    - <domain>-domain-skill
  output_schema: review-findings.schema.json
```

### Bước 2: Tạo agent file
Tạo `agents/review/<domain>-review-agent.md` theo template:
```markdown
---
name: <domain>-review-agent
description: >
  Khi nào dùng agent này, ví dụ trigger, output.
  KHÔNG dùng cho: <loại trừ rõ ràng>.
version: 0.1.0
---

# SKILL: <Domain> Review Agent

## 1. PURPOSE
## 2. WHEN TO USE
## 3. EXPECTED INPUTS
## 4. WORKFLOW
## 5. DOMAIN CHECKLIST
## 6. OUTPUT FORMAT
## 7. DEPENDENCIES
## 8. FINAL CHECK
```

### Bước 3: Tạo domain skill
Tạo `skills/domains/<domain>-domain-skill.md`:
```markdown
---
name: <domain>-domain-skill
description: >
  Tri thức domain cho <domain>.
version: 0.1.0
---

# SKILL: <Domain> Domain

## 1. PURPOSE
## 2. ARCHITECTURE
## 3. COMMON BUG PATTERNS      ← BẮT BUỘC có code examples
## 4. REVIEW HEURISTICS
## 5. DOMAIN-SPECIFIC RISKS
## 6. CHECKLIST
## 7. SEVERITY GUIDELINE
## 8. TEST GAPS TO FLAG
```

### Bước 4 (tùy chọn): Tách sub-skills nếu domain phức tạp

**Khi nào nên tách:** Domain có nhiều loại file / layer, checklist >100 dòng,
hoặc specialist knowledge khác biệt rõ giữa các phần.

**Pattern tham chiếu: `web-review-agent`**
```
web-review-agent                ← lean orchestrator
  └─ web-domain-skill           ← overview + routing
       ├─ web-html-skill        ← HTML/form/CSRF
       ├─ web-javascript-skill  ← JS/AJAX/XSS
       ├─ web-cgi-skill         ← CGI C/shell injection
       └─ web-luci-skill        ← OpenWrt LuCI/Lua
```

Agent chỉ phân loại file → delegate cho sub-skill phù hợp.
Thêm sub-skills vào `required_skills` của agent trong registry.

### Bước 5: Không sửa orchestrator
Orchestrator đọc registry + schema. Domain mới tuân thủ contract → hoạt động ngay.

---

## PHẦN 2: THÊM SKILL MỚI

### 2.1 Core skill (shared)
Ví dụ: `git-context-skill`, `embedded-c-code-review-skill`
- Tạo `skills/core/<name>-skill.md`
- Thêm vào `required_skills` của các agent cần dùng

### 2.2 Domain sub-skill
Ví dụ: `web-cgi-skill`, `web-luci-skill`
- Tạo `skills/domains/<name>-skill.md`
- Thêm vào `required_skills` của domain agent parent
- Agent reference sub-skill trong workflow (phân loại file → sub-skill nào)

### Template sub-skill
```markdown
---
name: <name>-skill
description: >
  Khi nào dùng, ví dụ trigger. Dùng bởi <parent-agent>.
version: 0.1.0
---

# SKILL: <Name>

## 1. PURPOSE
## 2. WHEN TO USE
## 3. COMMON BUG PATTERNS   (BẮT BUỘC có code examples)
## 4. REVIEW HEURISTICS
## 5. CHECKLIST
## 6. SEVERITY GUIDELINE
## 7. DEPENDENCIES
```

---

## PHẦN 3: SỬA ĐỔI AGENT / SKILL HIỆN CÓ

### 3.1 Sửa workflow/checklist
- Edit file `.md` tương ứng
- Bump `version` trong frontmatter
- Cập nhật changelog trong `docs/index.html`

### 3.2 Mở rộng `allowed_paths` / `triggers`
- Edit `agent-registry.yaml`
- Verify không conflict với agent khác (path/trigger overlap)

### 3.3 Thay đổi schema
- Edit `schemas/*.json`
- Phải backward-compatible HOẶC bump major version
- Update mọi reference schema

---

## PHẦN 4: CONTRACT BẮT BUỘC

### Sub-Agent Contract
- Input: `review-task.schema.json`
- Review chỉ trong `scope_files` được cấp
- Output: `review-findings.schema.json`
- Severity ∈ `{critical, major, minor, info}`
- Confidence ∈ `{high, medium, low}`
- Mỗi finding có: `evidence`, `impact`, `recommendation`, `confidence`

### Skill Contract
- Frontmatter YAML với `name`, `description`, `version`
- `description` rõ ràng về khi nào dùng và KHÔNG dùng

---

## PHẦN 5: NAMING CONVENTIONS

| Kind | Pattern | Ví dụ |
|------|---------|-------|
| Domain agent | `<domain>-review-agent` | `firewall-review-agent` |
| Domain skill | `<domain>-domain-skill` | `firewall-domain-skill` |
| Sub-skill | `<domain>-<aspect>-skill` | `web-cgi-skill` |
| Core skill | `<concern>-skill` | `git-context-skill` |
| Test agent (tương lai) | `<domain>-test-agent` | `dhcp-test-agent` |

---

## PHẦN 6: BEST PRACTICES

- **Domain = technical boundary thật**, không phải folder name
- **KHÔNG tạo agent overlap**: đã có `pppoe-review-agent` → đừng tạo `ppp-review-agent`
- **KHÔNG tạo agent "chung"** như `network-review-agent` — để `generic-review-agent` handle
- **Sub-skill hẹp**: mỗi sub-skill ~50-150 dòng, focus 1 aspect
- **Code examples** bắt buộc trong bug patterns
- **Severity guideline cụ thể per skill**
- **Always include** `embedded-c-code-review-skill` nếu agent review C code

---

## PHẦN 7: CHECKLIST TRƯỚC KHI COMMIT

- [ ] Registry entry YAML hợp lệ, không overlap path/trigger
- [ ] Agent file có đủ 8 section standard
- [ ] Skill file có bug patterns kèm code examples
- [ ] Version = 0.1.0 cho lần đầu
- [ ] Đã test với 1 review request thực tế
- [ ] Cập nhật `README.md` danh sách agents/skills
- [ ] Cập nhật `docs/index.html` (Agents/Skills/Changelog sections)

---

## PHẦN 8: VÍ DỤ — THÊM FIREWALL AGENT

**Mục tiêu:** Review iptables/nftables/fw3/fw4 rules

### 1. Registry entry
```yaml
- id: firewall-review-agent
  type: review-agent
  domain: firewall
  capabilities: [iptables, nftables, filter, nat, forward]
  triggers: [firewall, iptables, nftables, fw3, fw4, INPUT, FORWARD]
  allowed_paths:
    - package/**/firewall/**
    - package/**/fw3/**
    - package/**/fw4/**
    - etc/config/firewall
  required_skills:
    - embedded-c-code-review-skill
    - firewall-domain-skill
  output_schema: review-findings.schema.json
```

### 2. Agent file
`agents/review/firewall-review-agent.md` theo template Bước 2.

### 3. Domain skill
`skills/domains/firewall-domain-skill.md` với bug patterns:
- Default ACCEPT policy sau flush rules
- NAT rules duplicate port forward
- Zone assignment mismatch với interface
- fw3 → fw4 migration compatibility

### 4. (Optional) Sub-skills
Nếu domain lớn, có thể tách:
- `firewall-iptables-skill`
- `firewall-nftables-skill`
- `firewall-zone-skill`

### 5. Test
Gửi 1 review request với file trong `package/network/firewall/` — orchestrator tự detect.

---

## PHẦN 9: VÍ DỤ — THÊM SUB-SKILL CHO DOMAIN HIỆN CÓ

**Mục tiêu:** Thêm `wifi-driver-ath11k-skill` (chipset cụ thể)

### 1. Tạo sub-skill file
`skills/domains/wifi-driver-ath11k-skill.md` với bug patterns đặc thù ath11k.

### 2. Thêm vào `required_skills` của `wifi-driver-review-agent`
```yaml
required_skills:
  - embedded-c-code-review-skill
  - wifi-driver-domain-skill
  - wifi-driver-ath11k-skill   # mới
```

### 3. Update agent workflow
Trong `wifi-driver-review-agent.md`, thêm bước phân loại:
"Nếu file thuộc `drivers/**/ath11k/**` → apply `wifi-driver-ath11k-skill`"
