---
name: dhcp-domain-skill
description: >
  Skill này nên được dùng khi review hoặc phân tích logic DHCP,
  ví dụ: "DHCP lease review", "odhcpd", "dnsmasq option parsing".
  Output là domain knowledge và checklist DHCP.
version: 0.1.0
---

# SKILL: DHCP Domain

## PURPOSE
Tri thức domain cho DHCP server/client/relay/lease.

## COMMON BUG PATTERNS
- option length không validate
- lease update không nhất quán
- renew/rebind timer sai
- config reload làm mất state ngoài mong muốn
