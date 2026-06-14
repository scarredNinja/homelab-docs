---
date: '2026-04-29T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: 'infrastructure, networking, monitoring'
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - UniFi
  - Prometheus
  - unifi-poller
  - Networking
  - BugFix
---

# Session Notes — 2026-04-29 — UniFi Adoption & unifi-poller Setup

## Session Goal

Re-adopt UniFi AP to new controller on VLAN 60, wire up unifi-poller for
Prometheus metrics, confirm all prior session completions.

---

## Confirmed Completions (Prior Sessions)

- subvol-110 migration verified and ZFS dataset destroyed ✅
- Transmission/compose-vpn running, downloads landing in `/mnt/downloads/complete` ✅
- Traefik entrypoint NIC binding confirmed — internal services accessible on `:443` ✅
- Arr stack deployed (Sonarr, Radarr, Prowlarr) ✅
- Plex stack deployed ✅
- Sonarr/Radarr download client wired to Transmission ✅

---

## Completed This Session

### 1. UniFi AP Re-adoption

AP was on old flat `192.168.1.x` network with no path to controller at `10.0.60.42`.

**Resolution path:**
- Retrieved SSH credentials from UniFi UI → Settings → System → Device Authentication
- Reconfigured AP switch port to untagged VLAN 60 on Extreme switch
- Adoption stuck twice:
  - First: key mismatch from migrated controller config
  - Second: CGNAT blocking Ubiquiti WebRTC/STUN cloud relay (See Bug #48)
- Final fix: factory reset AP, disabled Remote Access in controller settings,
  set inform URL directly via SSH to `http://10.0.60.42:8080/inform`
- AP adopted cleanly

**SSID → VLAN mapping confirmed:**

| SSID | VLAN | Subnet |
|------|------|--------|
| Netbacon | 10 | 10.0.10.x |
| IoT | 20 | 10.0.20.x |
| Guest | 30 | 10.0.30.x |

### 2. unifi-poller — PR #14 Merged

| Commit | Change |
|---|---|
| `0ddc4d7` | unpoller service added to controller stack; Prometheus scrape at `tasks.controller_unpoller:9130` |
| `c73b2de` | UniFi router middleware fixed: `internal-only` → `internal-only@file` |
| `f231adb` | Deploy script: `middlewares.yaml` and `vpn-transmission.yml` now copied |

---

## Bugs Found

> [!bug] Bug #48 — UniFi adoption fails via Ubiquiti cloud relay behind CGNAT
> **Symptom:** Adoption stuck — controller logs show WebRTC/STUN/TURN failures
> to `162.159.207.0` and `141.101.90.1`
> **Cause:** Ubiquiti uses WebRTC cloud relay for adoption; CGNAT blocks required
> UDP punch-through
> **Fix:** Factory reset AP, set inform URL directly via SSH, disable Remote
> Access in controller Settings → System → Advanced
> **See Gotcha #48**

> [!bug] Bug #49 — Traefik middleware reference requires `@file` suffix
> **Symptom:** `unifi.home.purvishome.com` returns 404 despite router enabled
> **Cause:** Middleware referenced as `internal-only` — Traefik v3 requires
> `internal-only@file` for cross-file references
> **Fix:** Update middleware label to `internal-only@file` in router config
> **See Gotcha #49**

---

## Current State

| Item | Status |
|------|--------|
| UniFi AP adopted | ✅ |
| SSIDs broadcasting, IP assignment working | ✅ |
| unifi-poller deployed | ✅ PR #14 merged |
| `unifi.home.purvishome.com` | ✅ Resolved 2026-05-08 |
| unifi-poller Prometheus scrape | ✅ Confirmed 2026-05-08 — data flowing into Grafana 11311/11315/11312 |
| Grafana dashboards 11311/11315/11312 | ✅ Imported 2026-05-07 |

---

## Next Session Priorities

1. **Fix `unifi.home.purvishome.com` 404** — start with Traefik dashboard,
   check router/service state, then container logs
2. **Verify unifi-poller scrape** — `curl http://prometheus:9090/api/v1/targets`
   confirm `unifi-poller` shows `health: up`
3. **Import Grafana dashboards** — IDs 11311 (UAP), 11315 (Clients), 11312 (USW)
4. **Home VLAN → HA access** — pfSense rule VLAN 10 → `10.0.60.42:8123` +
   add `10.0.10.0/25` to Traefik `internal-only` middleware allowlist
5. **Grafana data source names** — update post-migration dashboard data sources

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Traefik]]
