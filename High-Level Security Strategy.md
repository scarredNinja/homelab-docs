---
project_id: Homelab-2025
phase: 'Phase 9: External Access'
tags:
  - Security
  - Networking
  - DMZ
  - Traefik
  - VPN
date: '2026-04-21T00:00:00.000Z'
status: Reference
service_status: deployed
---

# High-Level Security Strategy

> [!abstract] Overview
> Defence-in-depth model: network segmentation via VLANs, minimal public attack surface, VPN-first remote access, and Cloudflare as the only public ingress path.

---

## Core Principles

1. **VPN first** — all personal remote access goes through WireGuard on pfSense. Public exposure is the exception, not the default.
2. **Minimal public attack surface** — only Plex is directly internet-reachable. All admin/management services are internal-only.
3. **VLAN segmentation** — devices and services are isolated by trust level. VLANs cannot talk to each other unless explicitly allowed by pfSense rules.
4. **No direct inbound except port 443** — pfSense forwards 443 to Traefik only; all other inbound ports are blocked.
5. **Cloudflare as ingress only** — port 443 on WAN is restricted to Cloudflare IP ranges. Direct IP access is blocked.

---

## Network Trust Zones

| VLAN | Name | Trust Level | Purpose |
|---|---|---|---|
| 60 | Management/Infrastructure | High | Swarm nodes, Proxmox, pfSense management |
| 70 | Trusted Clients | High | Personal devices, VPN exit point |
| 50 | Media | Medium | Media VMs, Plex, Arr stack |
| 40 | General | Medium | General-purpose workloads |
| 80 | DMZ | Low | Traefik DMZ node — internet-facing |
| 100 | Storage | Isolated | NAS/NFS traffic only — no internet |
| 10 | IoT | Isolated | Smart home devices — no access to other VLANs |

---

## Public Attack Surface

| Service | Public? | Why | Protection |
|---|---|---|---|
| Plex | ✅ Yes | Requires internet for remote playback and account auth | Cloudflare IP restriction, own auth |
| All other services | ❌ No | VPN covers all remote access needs | VLAN 60/70 only |

> [!warning] Keep the public list short
> Every service exposed publicly is an ongoing maintenance and patching obligation. Default to VPN. Only expose when internet access is a hard requirement.

---

## Ingress Architecture

```
Internet
    │
    ├── Port 443 (Cloudflare IPs only via pfSense alias)
    │       └── pfSense NAT → traefik-dmz-01 VLAN 80 NIC
    │                               └── websecure entrypoint → Plex only
    │
    └── WireGuard VPN port (pfSense inbound)
            └── VPN client gets VLAN 70 equivalent access
                    └── traefik-dmz-01 VLAN 60 NIC → internal entrypoint
                            └── All other services
```

See [[Traefik Routing Architecture]] for full entrypoint and service routing details.

---

## Remote Access Strategy

| Use Case | Method | Notes |
|---|---|---|
| Portainer, Grafana, Proxmox UI | WireGuard VPN → internal Traefik | No public exposure needed |
| Sonarr/Radarr/Prowlarr | WireGuard VPN → internal Traefik | Admin tools, VPN is appropriate |
| Plex external streaming | Cloudflare → Traefik websecure | Only case requiring public port |
| Future external services | Authelia + websecure entrypoint | Deploy Authelia first — see [[Traefik Routing Architecture#Adding Authelia (Future)]] |

---

## Authentication Layers

| Layer | Technology | Status |
|---|---|---|
| Network perimeter | pfSense firewall rules + VLAN segmentation | ✅ Active |
| VPN access | WireGuard on pfSense | ✅ Active |
| Reverse proxy (external) | Traefik + Cloudflare IP restriction | ✅ Active |
| Application-level SSO | Authelia | 🔲 Planned (Phase 9) |

---

## Open Security Tasks

- [ ] Review `internal-only` middleware allowlist — `10.0.4.0/24` and `10.0.10.0/24` are stale entries that expand trust boundary [priority:: 1]
- [ ] look at rdp access for my pc [priority:: 1]at rdp access for my pc 
- [ ] Add `internal-only` middleware to InfluxDB router in `stack-monitoring.yml` (currently publicly routable on websecure) [priority:: 1]
- [ ] Audit pfSense VLAN 80 rules — confirm `Block VLAN80 → RFC1918` is ordered correctly after explicit allow rules [priority:: 2]
- [ ] Validate WireGuard VPN end-to-end — connect client, verify VLAN 70 access to internal Traefik services [priority:: 2]
- [ ] Evaluate Authelia deployment once VPN-only access proves insufficient [priority:: 3] #Later

---

## Related

- [[Traefik Routing Architecture]] — entrypoints, service routing, pfSense rules summary
- [[VLAN and Subnet Summary Sheet]] — full VLAN/subnet layout
- [[pfSense Firewall Rules]] — rule detail
- [[01 Homelab Rebuild - Phase 9 External Access Hub]] — Phase 9 tasks
