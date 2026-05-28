---
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - traefik
  - VLAN
---

> [!note] Superseded
> This file contains old planning notes and completed setup tasks. The canonical reference is [[Traefik Routing Architecture]]. The service record is [[Service - traefik]].

## Completed Setup (historical)

- [x] Setup reverse proxy ✅
- [x] HTTPS redirect and auth ✅
- [x] Setup pfSense ✅ 2026-02-14
- [x] Setup Portainer ✅ 2026-02-14
- [x] Setup PiHole ✅
- [x] Setup Proxmox ✅ 2026-02-14

## Decision: Single Instance, Two Entrypoints

~~Setup an internal proxy and a separate external proxy to handle external requests.~~
Resolved: single Traefik instance on `traefik-dmz-01`, two entrypoints on separate NICs. See [[Traefik Routing Architecture]] for rationale and full config.