---
date: '2026-04-24T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: Migration + Deployment
status: Completed
tags:
  - SessionNotes
  - DockerSwarm
  - Migration
  - HomeAssistant
  - UniFi
  - Plex
  - Arr
  - Monitoring
---

# Session Notes — 2026-04-24 — Final Service Migration & Stack Deployment

## 🎯 Session Goal

Complete final service migration from MainStorage LXC containers to Docker Swarm VMs. Provision `worker-controller-01`. Fix and deploy remaining stacks (arr, plex, controller, uptime-kuma).

---

## ✅ Completed

### Provisioned worker-controller-01

- VM provisioned: VLAN 60 (management) + VLAN 20 (IoT)
- IP: 10.0.60.42
- Labels applied: `zone=controller`, `nas=nfs`
- Joined Docker Swarm ✅
- virtiofs datasets mounted: docker-data, docker-db, docker-swarm ✅

---

### Service Config Migration from MainStorage LXC Containers

Imported `MainStorage` ZFS pool (`zpool import -f MainStorage`). Located all service data inside LXC subvolume filesystems. Paths were not standardised across containers — see Gotcha #39.

| Service | Source LXC | Source Path | Destination |
|---------|-----------|-------------|-------------|
| Sonarr | CT104 | `/MainStorage/subvol-104-disk-0/home/docker/sonarr/` | `/mnt/docker-data/sonarr/` |
| Radarr | CT104 | `/MainStorage/subvol-104-disk-0/home/docker/radarr/` | `/mnt/docker-data/radarr/` |
| Prowlarr | CT103 | `/MainStorage/subvol-103-disk-0/home/docker/prowlarr/` | `/mnt/docker-data/prowlarr/` |
| Transmission | CT103 | `/MainStorage/subvol-103-disk-0/home/docker/transmission/` | `/mnt/docker-data/transmission/` |
| Plex | CT101 | `/MainStorage/subvol-101-disk-0/home/docker/plex/` | `/mnt/docker-data/plex/` |
| Tautulli | CT101 | `/MainStorage/subvol-101-disk-0/home/docker/tautulli/` | `/mnt/docker-data/tautulli/` |
| InfluxDB | CT109 | `/MainStorage/subvol-109-disk-0/home/docker/influxdb/` | `/mnt/docker-tsdb/influxdb/` |
| Home Assistant | CT102 | `/MainStorage/subvol-102-disk-0/home/ha/docker/homeassistant/` | `/mnt/docker-data/homeassistant/` |
| UniFi | CT110 | `/MainStorage/subvol-110-disk-0/home/docker/unifi/` | `/mnt/docker-db/unifi/` |
| Uptime Kuma | CT110 | `/MainStorage/subvol-110-disk-0/home/docker/uptime-kuma/` | `/mnt/docker-data/uptime-kuma/` |
| Prometheus | CT109 | `/MainStorage/subvol-109-disk-0/home/docker/prometheus/` | `/mnt/docker-tsdb/prometheus/` |
| Grafana | CT109 | `/MainStorage/subvol-109-disk-0/home/docker/grafana/` | `/mnt/docker-data/grafana/` |

> [!important] Downloads transfer in progress
> 440 GB of downloads still transferring from CT103 `/home/` flat disk to `worker-mediamanagement-01:/mnt/downloads`.
> Sonarr/Radarr download client wiring deferred until transfer completes.

**Prometheus:** 5.4 GB solar history migrated. Post-rsync `chown -R 65534:65534` required — see Bug #27.

**Grafana:** Old dashboards migrated. Data sources need updating to new Swarm service names (Prometheus, InfluxDB).

**Plex:** Full delta rsync run (61 GB config/metadata). ADVERTISE_IP set in stack env. Dual custom URLs configured for both routers.

---

### virtiofs Permissions — Post-rsync chown Required (Bug #27)

After rsync from MainStorage, all data landed as `root:root`. virtiofs exposes host-side ZFS permissions directly — no UID remapping. Containers running as non-root UIDs failed to start.

```bash
# Applied on Proxmox host before starting containers
chown -R 65534:65534 /mnt/docker-tsdb/prometheus/   # Prometheus runs as nobody
chown -R 472:472 /mnt/docker-data/grafana/            # Grafana UID 472
chown -R 1000:1000 /mnt/docker-db/unifi/              # UniFi UID 1000
```

See Gotcha #36.

---

### Arr Stack — Gluetun Removed, Transmission Fixed (PR #8)

`network_mode: service:gluetun` silently ignored in Swarm mode (Bug #26). VPN kill switch moved to pfSense policy routing at the network layer — all traffic from `worker-mediamanagement-01` (10.0.50.51) routed through VPN interface in pfSense.

Stack changes:
- Gluetun service removed from `stack-arr.yml`
- Transmission `network_mode: service:gluetun` removed
- Transmission now joins overlay networks directly
- pfSense VLAN 50 policy routing alias updated to `10.0.50.51`

PR #8 merged ✅

See Gotcha #37.

---

### Controller Stack — Deployed (PR #9)

New stack `stacks/controller/stack-controller.yml` created and deployed on `worker-controller-01`.

**Home Assistant:**
- Config migrated from CT102
- Stack deployed, container starting
- Traefik file-provider route: `homeassistant.home.purvishome.com` → `http://10.0.60.42:8123`

**UniFi:**
- `lscr.io/linuxserver/unifi-controller` used (bundled MongoDB)
- Migration to `unifi-network-application` deferred — requires external MongoDB and data export/import cycle (see Gotcha #38)
- Config migrated from CT110 to `/mnt/docker-db/unifi/`
- chown 1000:1000 applied

PR #9 merged ✅

---

### Plex Stack — ADVERTISE_IP and Dual Router URLs

- `ADVERTISE_IP=http://10.0.50.50:32400` set in stack env
- Custom server access URLs set for both router IPs
- Relay disabled in Plex Settings → Network
- Library scan completed ✅

---

### Uptime Kuma Stack — Created, Not Yet Deployed

Stack file `stacks/monitoring/stack-uptime-kuma.yml` created. Placement: `zone=monitoring` (`worker-monitoring-01`).
Config migration from CT110 complete. Deployment deferred to next session.

---

## 🐛 Bugs

> [!bug] Bug #26 — `network_mode: service:gluetun` silently ignored in Docker Swarm
> **Symptom:** Transmission starts but VPN tunnelling is not applied. Gluetun and Transmission both have independent network namespaces. Other services cannot resolve `gluetun:9091`.
> **Root cause:** `network_mode: service:X` is a Docker Compose feature that shares network namespaces between containers on the same host. In Swarm mode this setting is silently ignored — each service always gets its own namespace.
> **Fix:** Remove Gluetun sidecar from Swarm stack. Use pfSense policy routing to route `worker-mediamanagement-01`'s traffic through VPN interface. Transparent to all containers, works correctly with Swarm networking.
> See Gotcha #37.

> [!bug] Bug #27 — virtiofs permissions after rsync (post-migration chown required)
> **Symptom:** Prometheus, Grafana, UniFi containers fail to start with `permission denied` on virtiofs-backed paths after rsync migration.
> **Root cause:** rsync copies files preserving source ownership (root:root from LXC container perspective). virtiofs exposes host-side ZFS permissions directly with no UID remapping.
> **Fix:** `chown -R <uid>:<gid>` on Proxmox host for each service before first container start. See Gotcha #36 for per-service UID reference.

> [!bug] Bug #28 — `unifi-network-application` requires external MongoDB
> **Symptom:** Container logs `*** No MONGO_HOST set ***` and fails to configure.
> **Root cause:** `lscr.io/linuxserver/unifi-network-application` requires a separate MongoDB container and `MONGO_HOST` env var. The older `unifi-controller` image bundles MongoDB internally.
> **Fix:** Use `lscr.io/linuxserver/unifi-controller` for now. Migration to `unifi-network-application` deferred — requires UniFi settings backup and restore cycle.
> See Gotcha #38.

---

## 📊 Current State

| Service | VM | Status | Notes |
|---------|-----|--------|-------|
| Sonarr | worker-mediamanagement-01 | ⏳ Pending | Config migrated, stack deployed. Download client not wired (downloads transfer in progress) |
| Radarr | worker-mediamanagement-01 | ⏳ Pending | Same as Sonarr |
| Prowlarr | worker-mediamanagement-01 | ✅ Running | Indexers operational |
| Transmission | worker-mediamanagement-01 | ✅ Running | VPN via pfSense policy routing — VLAN 50 → VPN |
| Plex | worker-media-01 | ✅ Running | Library scan complete, relay disabled, custom URLs set |
| Tautulli | worker-media-01 | ✅ Running | |
| Home Assistant | worker-controller-01 | 🟡 Deploying | Config migrated. Traefik file-provider route active. |
| UniFi | worker-controller-01 | 🟡 Deploying | unifi-controller (bundled MongoDB). Config migrated. |
| Uptime Kuma | worker-monitoring-01 | ⏳ Pending | Stack created, not yet deployed |
| Prometheus | worker-monitoring-01 | ✅ Running | Solar history migrated (5.4 GB). chown 65534 applied. |
| Grafana | worker-monitoring-01 | ✅ Running | Dashboards migrated. Data sources need updating. |
| InfluxDB | worker-monitoring-01 | ✅ Running | v2 data migrated from CT109 |

---

## ➡️ Next Session Priorities

1. Verify Home Assistant and UniFi containers fully running on `worker-controller-01`
2. Wire Sonarr/Radarr → Transmission download client once 440 GB transfer complete
3. Deploy Uptime Kuma stack
4. Update Grafana data sources to new Swarm service names
5. Update pfSense VLAN 50 VPN policy routing alias from old IP to 10.0.50.51
6. Check node-exporter/cAdvisor coverage — confirm all nodes appearing in Prometheus

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]
- [[Swarm Topology]]
- [[VM - worker-controller-01]]
