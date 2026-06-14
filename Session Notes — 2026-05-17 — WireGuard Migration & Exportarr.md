---
date: '2026-05-17T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm & Virtualisation'
session_type: implementation
status: Completed
tags:
  - vpn
  - gluetun
  - wireguard
  - monitoring
  - exportarr
  - arr
  - performance
---

# Session Notes — 2026-05-17 — WireGuard Migration & Exportarr

## Session Goal

Resolve Radarr/Sonarr UI load latency on `worker-mediamanagement-01` traced
to gluetun OpenVPN single-threaded CPU pin. Migrate gluetun to WireGuard
(NordLynx) for ~5× CPU efficiency. Add Exportarr Prometheus sidecars for
continuous visibility into request latency.

---

## Completed

### Gluetun OpenVPN → WireGuard (NordLynx) migration

`compose-vpn.yml` updated on `worker-mediamanagement-01`. OpenVPN environment
variables replaced with WireGuard equivalents (`VPN_TYPE=wireguard`,
`VPN_SERVICE_PROVIDER=nordvpn`, `WIREGUARD_PRIVATE_KEY`, `SERVER_COUNTRIES=New Zealand`,
`SERVER_CATEGORIES=P2P`, `FIREWALL_OUTBOUND_SUBNETS=10.0.0.0/8`).

Tunnel live post-deploy. Exit IP confirmed as a NordVPN NZ P2P node.
Root cause of CPU pin eliminated. See Gotcha #70.

> [!important] WireGuard private key is account-stable — extracted once from
> NordVPN client, stored in `compose-vpn.yml` on the virtiofs share (not in
> the public git repo).

### Exportarr sidecars added to arr stack

Two Exportarr sidecar services added to `stack-arr.yml`:
- `sonarr-exporter` — port 9707, scrapes Sonarr at `sonarr:8989`
- `radarr-exporter` — port 9708, scrapes Radarr at `radarr:7878`

Both join the `monitoring` overlay network so Prometheus on
`worker-monitoring-01` can scrape without a VLAN firewall rule.
API keys (`SONARR_API_KEY`, `RADARR_API_KEY`) are set as Portainer stack
env vars — not in YAML.

Prometheus scrape jobs added targeting `10.0.50.30:9707` and `10.0.50.30:9708`.

### arr stack fixes bundled in PR #39

- Fixed arr stack overlay network name (`monitoring_monitoring` → `monitoring`)
- Fixed Sonarr Traefik port label (was `9898`, correct is `8989`)
- Entrypoints confirmed already correct — no change needed

**PR #39** created on branch `feature/wireguard-migration-exportarr`.

---

## Current State

| Component | Status |
|-----------|--------|
| Gluetun WireGuard tunnel | ✅ Live — NZ P2P node confirmed |
| OpenVPN CPU pin | ✅ Eliminated |
| Exportarr sonarr-exporter (9707) | 🔲 Deployed — API key not yet set |
| Exportarr radarr-exporter (9708) | 🔲 Deployed — API key not yet set |
| Prometheus Exportarr scrape jobs | ✅ Added |
| Gotcha #70 | ✅ Added to runbook |

> [!bug] Radarr/Sonarr UI load latency post-redeploy
> After arr stack redeploy, direct port access refused and UI took 5+ minutes
> to load. Root cause unconfirmed — may be Exportarr container startup delay
> or overlay network re-registration. Investigate next session.

---

## Next Session Priorities

1. Set `SONARR_API_KEY` / `RADARR_API_KEY` in Portainer arr stack env vars — Exportarr comes online
2. Investigate post-redeploy UI latency (direct port access refused, 5+ min load)
3. Merge PR #39 after overnight validation

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix D Gotcha #70
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Service - radarr]]
- [[VM - worker-mediamanagement-01]]
