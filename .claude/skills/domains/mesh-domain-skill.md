---
name: mesh-domain-skill
description: >
  Skill này nên được dùng khi review Mesh stack/topology/roaming/backhaul,
  ví dụ: "mesh topology", "roaming logic", "controller-agent sync".
  Output là domain knowledge và checklist Mesh.
version: 0.1.0
---

# SKILL: Mesh Domain

## PURPOSE
Tri thức domain cho mesh topology, steering, controller-agent sync.

## COMMON BUG PATTERNS
- state sync không nhất quán giữa node
- backhaul selection logic thiếu recovery path
- config propagation không đủ
