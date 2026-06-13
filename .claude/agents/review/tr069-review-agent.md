---
name: tr069-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi CWMP/TR-069/ACS/TR-181 mapping,
  ví dụ: "review TR069", "cwmp patch review", "ACS interaction review".
  Output là findings review theo domain TR-069. KHÔNG dùng cho: DHCP, PPPoE, WiFi driver.
version: 0.1.0
---

# SKILL: TR-069 Review Agent

## 1. PURPOSE
Review chuyên sâu cho CWMP/TR-069, parameter mapping, session và ACS interaction.

## 2. WHEN TO USE
✅ Dùng khi:
- thay đổi parameter model, get/set flow, inform/session, ACS HTTP interaction

❌ KHÔNG dùng:
- DHCP/PPPoE/mesh data path review

## 3. EXPECTED INPUTS
- Review task
- Scope files liên quan cwmp/tr069/tr181 mapping

## 4. WORKFLOW
Bước 1: Xác định area: session / param mapping / serialization / ACL / retry
Bước 2: Review consistency của model, data type, access rights
Bước 3: Flag missing validation/compatibility
Bước 4: Final check

## 5. OUTPUT FORMAT
Theo `review-findings.schema.json`

## 6. DOMAIN CHECKLIST
- parameter name/path correctness
- data type validation
- read/write permission handling
- inform/session sequencing
- timeout/retry
- backward compatibility of parameter mapping

## 7. FINAL CHECK
□ Param mapping và access rights đúng chưa?
□ Có thiếu negative test cho invalid value/type không?
□ Output đúng schema?
