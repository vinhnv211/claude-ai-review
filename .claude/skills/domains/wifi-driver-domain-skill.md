---
name: wifi-driver-domain-skill
description: >
  Skill này nên được dùng khi review WiFi driver/cfg80211/mac80211/vendor driver,
  ví dụ: "driver callback", "assoc path", "channel config".
  Output là domain knowledge và checklist WiFi driver.
version: 0.1.0
---

# SKILL: WiFi Driver Domain

## PURPOSE
Tri thức domain cho driver WiFi và kernel interaction.

## COMMON BUG PATTERNS
- callback lifecycle sai
- lock/timer/memory ownership lỗi
- cfg80211/mac80211 API usage không đúng
