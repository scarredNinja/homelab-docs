---
date: '2026-04-23T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm — Media Stack'
session_type: Diagnostic + Fix
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Plex
  - Cloudflare
  - Traefik
  - Sonarr
  - CGNAT
  - pfSense
---

# Session Notes — 2026-04-23 — Plex External Access & Sonarr Fix

## 🎯 Session Goal
Diagnose and fix Plex external access via plex.purvishome.com, then investigate Sonarr 404.

---

## ✅ Completed

### Plex External Access — Full Diagnostic Path

Worked through all four layers systematically:

| Layer | Check | Result |
|-------|-------|--------|
| DNS | `Resolve-DnsName plex.purvishome.com` (PowerShell) | ✅ CNAME → ddns.purvishome.com → WAN IP |
| pfSense NAT | Port forward WAN:443 → traefik_dmz | ✅ Rule existed, source CloudFlare_IPs |
| Traefik | plex-external@swarm router | ✅ websecure entrypoint, enabled |
| Backend | wget from Traefik container → plex:32400 | ✅ Overlay connected |

External test from 5G mobile timed out despite all layers passing. Packet capture on
pfSense WAN interface revealed source IP `100.77.48.29` — in `100.64.0.0/10` CGNAT range.
ISP is NAT-ing before pfSense. No inbound port forwarding is possible. See Gotcha #33.

### Cloudflare Tunnel Deployed — Plex External Working ✅

Replaced pfSense port forward approach with Cloudflare Tunnel (cloudflared):
- Created tunnel in Cloudflare Zero Trust → Networks → Tunnels
- Public hostname configured: `plex.purvishome.com` → `http://plex:32400`
- Deployed `cloudflared` as Swarm service in `stack-plex.yml`
- Placement: `type=media` (worker-media-01)
- Networks: `media` overlay + `traefik-public`
- Token stored as `CLOUDFLARE_TUNNEL_TOKEN` env var
- Removed old grey-cloud CNAME `plex → ddns.purvishome.com`
- Tunnel creates its own orange-cloud CNAME automatically
- Plex accessible at `https://plex.purvishome.com` from external 5G ✅

> [!note] Cloudflare ToS
> Free plan technically prohibits large media streaming through their proxy. Plex is
> personal use / grey area. If tunnel is suspended, configure Plex direct stream mode.

> [!important] Plex settings — defer until after final rsync
> Plex Settings → Network → Custom server access URLs: `https://plex.purvishome.com:443`
> Disable Enable Relay. Do not change until final rsync is complete and validated.

---

### Sonarr 404 — Full Diagnostic Path

| Step | Finding |
|------|---------|
| Traefik router list | `sonarr@swarm` present, `websecure`, enabled |
| Service labels | `loadbalancer.server.port=9898` — typo (SslPort from config.xml, not Port) |
| Fixed port to 8989 in Portainer | Still 404 |
| Traefik service inspect | Stale IP — Traefik had cached old container IP |
| Container netstat | Sonarr listening on `:::8989` inside container ✅ |
| curl to container IP from host | Connection refused on overlay IP |
| `docker network inspect traefik-public --format '{{json .Peers}}'` on traefik-dmz-01 | `worker-mediamanagement-01` missing from peers |
| Same check on worker-mediamanagement-01 | Only sees itself — asymmetric |
| Ping traefik-dmz-01 → worker-mediamanagement-01 | ✅ Works |
| Ping worker-mediamanagement-01 → traefik-dmz-01 | ❌ Fails |
| Root cause | pfSense blocking UDP 4789 (VXLAN) from VLAN 50 → VLAN 60. VXLAN is bidirectional — one-way block prevents overlay mesh forming. See Gotcha #35. |
| Fix | Added pfSense rules VLAN50 → VLAN60: UDP 4789, TCP/UDP 7946, TCP 2377 |
| After fix | Peers populated on both nodes |
| `docker service update --force arr_sonarr` | New overlay IP assigned, Traefik picks it up |
| Entrypoint issue | Sonarr on `internal` entrypoint (8443) — LAN browser defaults to 443 |
| Fix | Changed sonarr/radarr/prowlarr entrypoints to `websecure` in Portainer |
| Result | `https://sonarr.home.purvishome.com` accessible ✅ |

See Gotcha #34 (port typo) and Gotcha #35 (VXLAN inter-VLAN).

---

## 🐛 Bugs

> [!bug] Bug #33 — CGNAT blocks all inbound port forwarding
> **Symptom:** External connections to plex.purvishome.com time out. All pfSense
> port forward rules have 0 states / 0 bytes matched. Packet capture on WAN shows
> source IP in `100.64.0.0/10` range.
> **Root cause:** ISP uses Carrier-Grade NAT. pfSense WAN gets a private `100.x.x.x`
> address. `118.148.x.x` is the ISP's shared public NAT gateway — inbound traffic
> never reaches pfSense.
> **Fix:** Cloudflare Tunnel — makes outbound connection to Cloudflare edge, bypasses
> CGNAT entirely. No open inbound ports required.

> [!bug] Bug #34 — Sonarr Traefik loadbalancer port typo
> **Symptom:** `sonarr.home.purvishome.com` returns 404. Traefik shows service UP
> but Sonarr connection refused on the backend IP.
> **Root cause:** `traefik.http.services.sonarr.loadbalancer.server.port=9898` —
> 9898 is the `<SslPort>` field in Sonarr's config.xml, not the `<Port>` field (8989).
> Digits transposed. Traefik health check VIP passes but routes to wrong port.
> **Fix:** Correct label to `8989` in stack-arr.yml. Always verify port against
> `netstat -tlnp` inside the running container, not config field names.

> [!bug] Bug #35 — Swarm VXLAN overlay fails between nodes on different VLANs
> **Symptom:** `docker network inspect traefik-public --format '{{json .Peers}}'`
> shows `worker-mediamanagement-01` (VLAN 50) missing from peers on `traefik-dmz-01`
> (VLAN 60). Node only sees itself. Overlay traffic fails despite service being
> attached to the network.
> **Root cause:** pfSense blocking UDP 4789 (VXLAN) inter-VLAN from VLAN 50 → VLAN 60.
> VXLAN requires bidirectional UDP 4789 between all Swarm node host IPs. Also requires
> TCP/UDP 7946 (gossip) and TCP 2377 (control plane).
> **Fix:** Add pfSense firewall rules on VLAN 50 interface: UDP 4789, TCP/UDP 7946,
> TCP 2377 → VLAN 60. After rules applied, force update services to trigger re-registration:
> `docker service update --force <service>`.

---

## 📊 Current State

| Service | URL | Status |
|---------|-----|--------|
| Plex (external) | https://plex.purvishome.com | ✅ Working via Cloudflare Tunnel |
| Plex (internal) | https://plex.home.purvishome.com | ✅ Working |
| Sonarr | https://sonarr.home.purvishome.com | ✅ Working |
| Radarr | https://radarr.home.purvishome.com | ✅ Working |
| Prowlarr | https://prowlarr.home.purvishome.com | ✅ (entrypoint fix applied, not verified) |

**Pending — Claude Code session:**
- stack-arr.yml: commit port 8989 fix, entrypoints internal→websecure, arr_default subnet pinned to 10.200.5.0/24
- stack-plex.yml: commit cloudflared service definition

**Pending — after final rsync:**
- Plex Settings → Network → Custom server access URLs: `https://plex.purvishome.com:443`
- Disable Enable Relay in Plex

---

## ➡️ Next Session Priorities

1. Run Claude Code session — commit stack-arr.yml and stack-plex.yml changes to repo
2. Configure Prowlarr indexers and sync to Sonarr/Radarr
3. Wire Transmission as download client in Sonarr/Radarr (gluetun:9091)
4. Final rsync validation, then Plex network settings
5. Grafana data sources and dashboards

---

## 🔗 Related Notes
- [[Docker Swarm Infrastructure Runbook]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
