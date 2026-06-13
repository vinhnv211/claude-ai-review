---
name: mesh-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi Mesh topology/backhaul/handoff/controller-agent,
  ví dụ: "review mesh stack", "802.11s review", "backhaul selection review", "roaming logic review".
  Output là findings review theo domain Mesh. KHÔNG dùng cho: DHCP server, PPPoE, Web UI rendering.
version: 0.1.0
---

# SKILL: Mesh Review Agent

## 1. PURPOSE
Review chuyên sâu cho Mesh stack, controller/agent interaction, backhaul, topology, handoff.

## 2. WHEN TO USE
✅ Dùng khi:
- thay đổi về topology, steering, backhaul selection, controller-agent sync, onboarding

## 3. EXPECTED INPUTS
- Review task
- Scope files liên quan mesh stack/802.11s/steering/control logic

## 4. WORKFLOW
Bước 1: Xác định area: topology/handoff/backhaul/config sync
Bước 2: Review state/event flow và consistency đa node
Bước 3: Ghi findings + missing roaming/topology tests
Bước 4: Final check

## 5. OUTPUT FORMAT
Theo `review-findings.schema.json`

## 6. DOMAIN CHECKLIST
- topology consistency
- controller-agent sync
- backhaul selection logic
- handoff/roaming decision path
- config propagation
- recovery after node flap

## 7. FINAL CHECK
□ Có issue về state sync/topology/handoff không?
□ Có thiếu multi-node test không?
□ Output đúng schema?
