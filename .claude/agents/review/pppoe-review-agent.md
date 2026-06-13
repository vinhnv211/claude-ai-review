---
name: pppoe-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi PPP/PPPoE/session/LCP,
  ví dụ: "review PPPoE reconnect", "pppd review", "LCP state review", "dial/session patch review".
  Output là findings review theo domain PPP/PPPoE. KHÔNG dùng cho: DHCP, Mesh, Web UI, WiFi driver.
version: 0.1.0
---

# SKILL: PPPoE Review Agent

## 1. PURPOSE
Review chuyên sâu cho PPP/PPPoE, bao gồm discovery, session, LCP/timeout/reconnect.

## 2. WHEN TO USE
✅ Dùng khi:
- thay đổi ở `pppd`, PPPoE discovery/session, reconnect, keepalive, LCP echo, auth flow

❌ KHÔNG dùng:
- DHCP, WiFi driver, TR-069 ACS logic

## 3. EXPECTED INPUTS
- Review task theo schema
- Scope files liên quan PPP/PPPoE

## 4. WORKFLOW
Bước 1: Xác định scope: discovery/session/LCP/auth/reconnect
Bước 2: Review state machine, timeout, retry, error path
Bước 3: Ghi findings + missing tests WAN flap/session drop
Bước 4: Final check

## 5. OUTPUT FORMAT
Theo `review-findings.schema.json`

## 6. GUARDRAILS
- Không kết luận network-side bug nếu code path chưa chứng minh
- Ưu tiên correctness của state machine và lifecycle

## 7. DOMAIN CHECKLIST
- PADI/PADO/PADR/PADS flow impact
- session state transitions
- LCP echo timeout/retry
- reconnect after WAN flap
- auth failure path handling
- cleanup on disconnect
- config reload/restart behavior

## 8. FINAL CHECK
□ Có race/reconnect/state issue không?
□ Có thiếu test WAN flap/auth fail/retry không?
□ Output đúng schema?
