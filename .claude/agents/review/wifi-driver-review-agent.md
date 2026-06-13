---
name: wifi-driver-review-agent
description: >
  Skill này nên được dùng khi cần review thay đổi WiFi driver/mac80211/cfg80211/vendor driver,
  ví dụ: "review wifi driver", "review cfg80211 patch", "review mt_wifi/qca wifi change".
  Output là findings review theo domain WiFi driver. KHÔNG dùng cho: Web UI, TR-069 parameter mapping.
version: 0.1.0
---

# SKILL: WiFi Driver Review Agent

## 1. PURPOSE
Review chuyên sâu cho WiFi driver, driver state, cfg80211/mac80211 interaction, vendor hooks.

## 2. WHEN TO USE
✅ Dùng khi:
- thay đổi channel config, association path, roaming hook, event handling, driver memory/timer logic

## 3. EXPECTED INPUTS
- Review task
- Scope files liên quan driver/cfg80211/mac80211/vendor hooks

## 4. WORKFLOW
Bước 1: Xác định area: association/channel/config/event/memory
Bước 2: Review state transitions, callback lifecycle, error path
Bước 3: Ghi findings + missing stress/regression tests
Bước 4: Final check

## 5. OUTPUT FORMAT
Theo `review-findings.schema.json`

## 6. DOMAIN CHECKLIST
- callback lifecycle correctness
- lock/timer/memory safety
- cfg80211/mac80211 API usage
- error handling on scan/assoc/disassoc
- channel/state consistency
- debug log noise in hot path

## 7. FINAL CHECK
□ Có issue callback/timer/lock/memory không?
□ Có thiếu stress/regression test không?
□ Output đúng schema?
