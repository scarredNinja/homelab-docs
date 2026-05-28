---
date: 2026-05-07
project_id: Homelab-2025
phase: "Phase 9: External Access"
session_type: Code + Implementation
status: Complete
tags:
  - Tailscale
  - VPN
  - AdminAccess
  - pfSense
  - Traefik
  - CGNAT
---

# Session Notes — 2026-05-07 — Tailscale Admin VPN

## Session Goal

Deploy Tailscale as an admin VPN to enable off-site access to VLAN 60 services without Cloudflare Tunnel. Required because ISP CGNAT (pfSense WAN = 100.77.x.x) makes inbound port forwarding impossible, and Cloudflare ToS prohibits using the tunnel for general admin traffic.

---

## Architecture

```
Admin device (anywhere)
    ↓ Tailscale client
Tailscale DERP relay (CGNAT traversal)
    ↓ encrypted tunnel
pfSense (Tailscale package, subnet router)
    ↓ subnet route: 10.0.60.0/25
VLAN 60 services (Traefik, Portainer, Grafana, etc.)
```

**Key components:**
- pfSense installs Tailscale package — acts as subnet router, no separate VM needed
- Tailscale ACL restricts access to `10.0.60.0/25` only
- Traefik `internal-only` middleware updated to allow Tailscale IP range `100.64.0.0/10`
- DERP relay handles CGNAT traversal — no inbound ports required

---

## Completed

### Traefik Middleware Update — Deployed

`config/traefik/dynamic/middlewares.yaml` updated in repo — added `100.64.0.0/10` to `internal-only` IPAllowList sourceRange. Already deployed.

### Tailscale on pfSense

- Tailscale package installed via pfSense Package Manager
- pfSense registered to Tailscale account
- Tailscale IP: `100.x.x.x` (pfSense node)
- Subnet route `10.0.60.0/25` advertised
- Route approved in Tailscale admin console

### pfSense Firewall Rule

- Rule added: Tailscale interface → VLAN 60 (10.0.60.0/25), allow all

### Client Enrolment

- Phone enrolled in Tailscale, subnet route enabled on client

### Repo Docs

- `docs/admin-vpn.md` created in [scarredNinja/docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home) — architecture, implementation checklist, security notes, upgrade path

---

## ✅ Resolved — ERR_ADDRESS_UNREACHABLE from Phone

### Symptom

Phone connected to Tailscale, subnet route active, browsing to `https://traefik.home.purvishome.com` (10.0.60.x via DNS) returns `ERR_ADDRESS_UNREACHABLE`.

### Investigation Done

- Tailscale dashboard: pfSense node online, route approved, phone showing subnet active
- pfSense firewall rule: Tailscale → LAN/VLAN 60 — rule visible in rules list
- `curl https://10.0.60.x` from phone — also fails (rules out DNS issue)

### Likely Causes (check next session)

1. **IP forwarding not enabled on pfSense** — `sysctl net.inet.ip.forwarding` should be `1`; may need explicit enable under System → Advanced → Networking
2. **pfSense outbound NAT** — Tailscale traffic from `100.64.x.x` may not be NATted to VLAN 60 source; pfSense needs outbound NAT rule or subnet route traffic handled differently
3. **Firewall rule not matching** — Tailscale interface in pfSense may use a different interface name; check pfSense → Interfaces → Tailscale binding

### Next Session Diagnostic Steps

```bash
# From phone (Tailscale connected):
ping 10.0.60.1   # pfSense VLAN 60 gateway — if this fails, routing not working

# On pfSense:
sysctl net.inet.ip.forwarding   # must be 1
# Check: Firewall → Logs → Firewall → filter for 100.64 source
# Check: Status → Gateways → all green?

# On manager-01 (10.0.60.30):
# Check if traffic reaches the node at all:
tcpdump -i any host 100.64.x.x
```

---

## Current State

| Component | Status |
|---|---|
| Tailscale on pfSense | ✅ Installed, subnet route advertised |
| Subnet route approved (admin console) | ✅ |
| pfSense firewall rule | ✅ Added |
| Traefik `internal-only` + Tailscale range | ✅ Deployed |
| Phone → VLAN 60 routing | ✅ Resolved 2026-05-08 |
| `docs/admin-vpn.md` in repo | ✅ Created |

---

## Next Session Priorities

- [x] Diagnose phone routing ✅ 2026-05-08
- [x] Tailscale deployed and routing confirmed ✅ 2026-05-08
- [ ] Configure Tailscale ACL: restrict to `10.0.60.0/25` only [priority:: 2]
- [ ] Enrol remaining admin devices (laptop) [priority:: 2]

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Phase 9, Appendix D
- [[01 Homelab Rebuild - Phase 9 External Access Hub]]
- [[Home Dashboard — Research & Build Plan]]
