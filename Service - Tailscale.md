---
type: pfsense-service
project_id: Homelab-2025
phase: "Phase 9: External Access"
tags:
  - Tailscale
  - VPN
  - AdminAccess
  - pfSense

service_name: Tailscale
vm: pfSense (not a Swarm service)
service_status: running
deployment: pfSense package (System → Package Manager)

last_updated: 2026-05-08
---

# Tailscale — Admin VPN

Tailscale runs on pfSense as a subnet router, providing off-site admin access to VLAN 60 services. Required because ISP WAN is CGNAT (`100.77.x.x`) — no inbound port forwarding is possible.

## Deployment

| Property | Value |
|---|---|
| Installed on | pfSense — System → Package Manager → Tailscale |
| Mode | Subnet router |
| Subnet advertised | `10.0.60.0/25` (VLAN 60 / Infrastructure) |
| Tailscale IP range | `100.64.0.0/10` |
| DERP relay | Active — handles CGNAT traversal, no inbound ports needed |
| Status | ✅ Running 2026-05-08 |

## Architecture

```
Admin device (anywhere)
    ↓ Tailscale client
Tailscale DERP relay  ← CGNAT traversal
    ↓ encrypted tunnel
pfSense (subnet router)
    ↓ 10.0.60.0/25
VLAN 60 services (Traefik, Portainer, Grafana, etc.)
```

## Traefik Integration

`config/traefik/dynamic/middlewares.yaml` — `100.64.0.0/10` added to `internal-only` IPAllowList sourceRange. Deployed ✅ 2026-05-07.

All services behind the `internal` entrypoint are reachable via Tailscale without any additional config.

## Enrolled Devices

| Device | Status |
|---|---|
| Phone | ✅ Enrolled, subnet route active |
| Laptop | ⏳ Pending |

## pfSense Config

- **Firewall rule:** Tailscale interface → VLAN 60 (`10.0.60.0/25`), allow all
- **Subnet route:** Advertised `10.0.60.0/25`, approved in Tailscale admin console
- **ACL policy:** Restrict to `10.0.60.0/25` only — pending configuration in Tailscale admin console

## Open Items

- [ ] Configure Tailscale ACL policy: restrict device access to `10.0.60.0/25` only, block IoT/home VLANs [priority:: 2]
- [ ] Enrol laptop [priority:: 2]

## Upgrade Path

| Option | When |
|---|---|
| Headscale | If Tailscale pricing/ToS changes — self-hosted coordination server |
| Authelia + passkeys | Phase 10 — application-level auth on top of Tailscale |

## Related

- [[Session Notes — 2026-05-07 — Tailscale Admin VPN]]
- [[Docker Swarm Infrastructure Runbook]] — Phase 9
- [[01 Homelab Rebuild - Phase 9 External Access Hub]]
- `docs/admin-vpn.md` — in [scarredNinja/docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home)
