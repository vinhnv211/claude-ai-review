---
name: dhcp-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi liên quan DHCP server/client/lease/config,
  ví dụ: "review DHCP change", "odhcpd review", "dnsmasq patch review", "lease handling review".
  Output là findings review theo domain DHCP. KHÔNG dùng cho: PPPoE, Mesh, WiFi driver, Web UI.
version: 0.1.0
---

# SKILL: DHCP Review Agent

## 1. PURPOSE
Review chuyên sâu cho logic DHCP trong embedded/OpenWrt.

## 2. WHEN TO USE
✅ Dùng khi:
- thay đổi ở `dnsmasq`, `odhcpd`, DHCP option parsing, lease management, renew/rebind, config mapping

❌ KHÔNG dùng:
- PPP negotiation, TR-069, mesh steering, web rendering

## 3. EXPECTED INPUTS
- Review task theo schema
- Scope files liên quan DHCP
- Optional: design note / bug context

## 4. WORKFLOW
Bước 1: Xác định DHCP area: server/client/relay/lease/config
Bước 2: Review theo checklist domain + core skills
Bước 3: Ghi findings với severity + evidence + impact
Bước 4: Chỉ ra missing tests liên quan discover/offer/request/ack, renew, invalid option
Bước 5: Final check

## 5. OUTPUT FORMAT
Theo `review-findings.schema.json`

## 6. GUARDRAILS
- Chỉ review trong scope được cấp
- Không suy diễn behavior ngoài context
- Không kết luận protocol bug nếu không có evidence từ code path

## 7. DOMAIN CHECKLIST
- option length validation
- lease lifetime consistency
- renew/rebind timers
- invalid/malformed packet handling
- config reload compatibility
- memory ownership of option buffers
- logging/noisy path

## 8. FINAL CHECK
□ Có issue về parsing/lease/timer/config không?
□ Có missing tests về renew/rebind/invalid option không?
□ Output đúng schema?
