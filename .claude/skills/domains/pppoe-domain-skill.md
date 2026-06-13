---
name: pppoe-domain-skill
description: >
  Skill này nên được dùng khi review hoặc phân tích logic PPP/PPPoE,
  ví dụ: "PPPoE reconnect", "LCP timeout", "pppd state".
  Output là domain knowledge và checklist PPP/PPPoE.
version: 0.1.0
---

# SKILL: PPPoE Domain

## PURPOSE
Tri thức domain cho PPP/PPPoE discovery/session/LCP/reconnect.

## COMMON BUG PATTERNS
- state transition không đầy đủ
- reconnect sau WAN flap không cleanup session cũ
- timeout/retry logic không nhất quán
- auth fail path không clear state
