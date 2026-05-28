---
status: Archived
project_id: Homelab-2025
phase: "Phase 6: Service Deployment"
archived_date: 2026-04-25
archive_reason: Merged into Phase 5 — all service deployment tracked there
---

> [!warning] Archived 2026-04-25
> Phase 6 goals were completed as part of [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]].
> All service status, deployment tasks, and configuration are tracked in Phase 5 notes and individual `Service - *.md` files.
> PBS tasks moved to [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]].

# 📅 Phase 6: Service Deployment (Archived)

## 🎯 Goals — Status

* ~~Deploy all planned core services~~ ✅ Completed in Phase 5 (Plex, Tautulli, Sonarr, Radarr, Prowlarr, Transmission, HA, UniFi, Prometheus, Grafana, InfluxDB, Loki, Uptime Kuma)
* ~~Configure internal DNS and reverse proxy access for all services~~ ✅ Traefik deployed in Phase 5 with wildcard TLS and internal/external entrypoints
* ~~Verify all persistent volumes are mapped correctly~~ ✅ virtiofs datasets confirmed, see [[Mount Point Map]]

## 🔗 Action Items — Superseded

### Infrastructure Services

- [x] **Deploy Proxmox Backup Server (PBS):** Moved to [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]] ✅ 2026-04-25

### Application Services

- [x] Update list of work services and areas/VLANs ✅ 2026-02-05
- [x] Get a copy of all service stacks and save to GIT ✅ Repo: scarredNinja/docker-swarm-home
- [x] **Deploy Media Services (VLAN 50):** Plex, Transmission deployed ✅ Phase 5
- [x] **Deploy Media Management (VLAN 50):** Sonarr, Radarr, Prowlarr deployed ✅ Phase 5
- [x] **Deploy Homelab Services:** Home Assistant, Grafana deployed ✅ Phase 5
- ~~Deploy Database Services (VLAN 65)~~ — superseded; no VLAN 65 in final topology
- ~~Deploy DMZ Services (VLAN 80) — Dashy~~ — superseded; not in final architecture
- [x] Create Docker Compose stacks for all core services ✅ 2026-02-15
- [x] Set up Traefik/Nginx Proxy Manager for internal routing ✅ 2026-02-05
- [x] Test application access via internal hostnames ✅ 2026-02-05
