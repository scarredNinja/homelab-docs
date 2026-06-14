---
date: '2026-04-12T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm — Phase 1.5 Monitoring'
tags:
  - SessionNotes
  - DockerSwarm
  - Monitoring
  - Prometheus
  - Grafana
  - Loki
  - InfluxDB
  - overlay2
  - BugFix
session_type: Code + Planning
status: Completed
---

# Session Notes — 2026-04-12 (Monitoring Deploy)

## 🎯 Session Goal

Deploy the monitoring stack (`worker-monitoring-01`), complete the `overlay2` storage migration from virtiofs-backed `fuse-overlayfs`, and produce next-session reference docs.

---

## ⚠️ Incident — worker-monitoring-01 Did Not Auto-Join Swarm

`worker-monitoring-01` was provisioned but did **not** automatically join the Swarm via `03-post-boot.sh`. The join and label application had to be performed manually.

**Manual recovery steps applied:**

```bash
# On worker-monitoring-01
docker swarm join --token <worker-token> 10.0.60.30:2377

# On manager-01 — apply zone label
docker node update --label-add zone=monitoring \
  $(docker node ls --filter name=worker-monitoring-01 -q)

# Verify
docker node inspect worker-monitoring-01 --pretty | grep -A5 Labels
```

**Root cause:** Auto-join logic in `03-post-boot.sh` did not verify join success, and label application post-join was missing or not triggered. This is captured in the Claude Code prompt issued this session for investigation and fix.

> [!bug] Bug #8 — Auto-join silent failure
> `03-post-boot.sh` does not verify that `docker swarm join` succeeded, and does not apply node labels on the manager after join. Fix: add join success check and `docker node update --label-add` step to the script. Claude Code task raised.

---

## 💻 Code Session — Merged to Main

All changes merged to `main`. Branch deleted.

### Files Changed

| File | Change |
|------|--------|
| `03-post-boot.sh` | `overlay2` migration complete; default disk bumped to 40G |
| `base-vendor.yaml` | Updated to match overlay2 data-root (`/var/lib/docker`) |
| `01-create-template.sh` | Default disk size bumped to **40G** |
| `stacks/monitoring/stack-monitoring.yml` | New — full monitoring stack (Prometheus, Grafana, Loki, Promtail, node-exporter, cAdvisor, InfluxDB) |
| `docs/swarm-recovery.md` | New — full swarm recovery runbook with §0 overlay2 migration |
| `docs/monitoring-next-steps.md` | New — Grafana data sources, dashboard IDs, TSDB migration notes |
| `stacks/stack-grafana.yml` | **Deleted** — stale draft |
| `stacks/stack-prometheus.yml` | **Deleted** — stale draft |

---

## 🔧 Key Change — overlay2 Migration

Docker's data-root moved from virtiofs (`/mnt/docker-data`) to local VM disk (`/var/lib/docker`). This eliminates the root cause of fuse-overlayfs layer corruption and Raft loss on Proxmox host restart.

**What moved:**
- Image layers, container state, Raft DB → `/var/lib/docker` (local, overlay2)
- Config bind mounts (Traefik certs, Prometheus config, etc.) → remain on `/mnt/docker-data`

**Why this matters:**
- Docker is no longer dependent on virtiofs being ready at startup for image/Raft state
- `wait-for-virtiofs.conf` systemd drop-in is still needed for config file availability, but it is no longer a hard boot dependency for the Docker daemon itself
- Swarm recovery after host restart is now significantly more reliable

**`daemon.json` post-migration:**
```json
{
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Pre-start cleanup drop-in updated** (path changed from virtiofs to local):
```ini
# /etc/systemd/system/docker.service.d/pre-start-cleanup.conf
[Service]
ExecStartPre=-/sbin/ip link set docker0 down
ExecStartPre=-/sbin/ip link delete docker0
ExecStartPre=-/sbin/ip link set docker_gwbridge down
ExecStartPre=-/sbin/ip link delete docker_gwbridge
ExecStartPre=-/bin/rm -f /var/lib/docker/network/files/local-kv.db
```

**Migration order for existing nodes** (workers first, manager last):
```
traefik-dmz-01       10.0.60.40  ← first
worker-monitoring-01 10.0.60.41  ← second
manager-01           10.0.60.30  ← last
```

---

## 📦 Monitoring Stack — Design Decisions

### Services

| Service | Mode | Placement | Mount |
|---------|------|-----------|-------|
| Prometheus | replicated (1) | `zone=monitoring` | TSDB → `/mnt/docker-tsdb/prometheus` |
| Grafana | replicated (1) | `zone=monitoring` | Data → `/mnt/docker-data/grafana/data` |
| Loki | replicated (1) | `zone=monitoring` | Data → `/mnt/docker-data/loki/data` |
| InfluxDB | replicated (1) | `zone=monitoring` | TSDB → `/mnt/docker-tsdb/influxdb` |
| Promtail | global | all nodes | `/var/lib/docker/containers` read-only |
| node-exporter | global | all nodes | `network_mode: host` |
| cAdvisor | global | all nodes | Docker socket + rootfs read-only |

### Important notes

- **node-exporter uses `network_mode: host`** — required for accurate host-level metrics. It cannot be on the overlay network. This is why it doesn't appear in `docker service ls` overlay network membership.
- **InfluxDB on `traefik-public`** — added so Traefik can route to the InfluxDB UI without a separate ingress stack.
- **Global services have restart delays** — `delay: 10s` added to prevent thundering-herd on swarm rebuild.
- **TSDB data survived the swarm rebuild on virtiofs** — Prometheus and InfluxDB both resumed from existing data. No migration needed.
- **`DOCKER_INFLUXDB_INIT_MODE: setup` is ignored** when the data directory is non-empty — this is correct behaviour.

### Image versions pinned

```
prom/prometheus:v2.51.2
grafana/grafana:11.0.0
grafana/loki:2.9.6
grafana/promtail:2.9.6
gcr.io/cadvisor/cadvisor:v0.49.1
prom/node-exporter:v1.7.0
influxdb:2.7   ← change from :latest recommended
```

> [!todo] Pin `influxdb:2.7` in `stack-monitoring.yml` — currently still `influxdb:latest`. Check current version first: `docker exec $(docker ps -q -f name=monitoring_influxdb) influx version`

---

## 📊 Current Swarm State (End of Session)

| Node | Role | Status | VLAN |
|------|------|--------|------|
| `manager-01` (10.0.60.30) | Leader | Ready | 60 |
| `traefik-dmz-01` (10.0.60.40) | Worker | Ready | 80+60 |
| `worker-monitoring-01` (10.0.60.41) | Worker | Ready | 60 |

| Stack | Status |
|-------|--------|
| `traefik` | Deployed ✅ |
| `portainer` | Deployed ✅ |
| `monitoring` | Deployed ✅ |

---

## 📋 Next Session Checklist (Grafana Setup)

See `monitoring-next-steps.md` for full details.

1. Log into `https://grafana.home.purvishome.com`
2. Add Prometheus data source → `http://prometheus:9090`
3. Add Loki data source → `http://loki:3100`
4. Add InfluxDB data source → `http://influxdb:8086`, Flux query language, org `home`, bucket `proxmox_metrics`
5. Import starter dashboards:

| Dashboard | Grafana ID | Source |
|-----------|-----------|--------|
| Node Exporter Full | `1860` | Prometheus |
| cAdvisor | `14282` | Prometheus |
| Docker Swarm | `609` | Prometheus |
| Loki logs | `13639` | Loki |

6. Verify Prometheus targets page — all global exporters should show `UP`

---

## ➡️ Next Sessions (In Order)

### 1. Grafana configuration (short — ~30 min)
Add data sources and import dashboards as above. Verify all scrape targets are green.

### 2. NetBox — Infrastructure documentation
Document the full network and VM topology before it gets more complex. Captures IP assignments, VLAN assignments, VM inventory, and service placement in a searchable, queryable format. Prevents drift between reality and documentation as more VMs are provisioned.

### 3. Plex + Arr — Split VM architecture

The media stack is split across two VMs. This was clarified this session.

#### worker-media-01 — Plex only (consumption)

- VLAN 50, `type=media` label
- 4 vCPU / 8GB RAM
- Runs: **Plex + Tautulli only**
- Mounts NFS share from Proxmox host at `/mnt/media` — **read-only**
- Plex config on virtiofs `/mnt/docker-data/plex/config`
- Transcode dir: `/tmp/transcode` inside container (RAM-backed tmpfs, not NFS or virtiofs)
- Stack file: `stacks/plex/stack-plex.yml`, placement `type=media`

#### worker-general-01 — Media management (Arr + downloads)

- VLAN 40, `zone=mediamanagement` label
- 4 vCPU / 6GB RAM
- Runs: **Sonarr, Radarr, Prowlarr, Transmission+Gluetun**
- Primary disk: 40G (OS + Docker + app configs on virtiofs)
- **Second disk**: 500GB–1TB ZFS dataset mounted at `/mnt/downloads` — download staging only
- Also mounts NFS share at `/mnt/media` — **read-write** (Sonarr/Radarr write the final file here)
- Stack file: `stacks/arr/stack-arr.yml`, placement `zone=mediamanagement`

#### Data flow (one copy, no hardlinks)

```
Transmission downloads → /mnt/downloads (local disk, worker-general-01)
         │
         ▼  Sonarr/Radarr import trigger (on completion)
Sonarr/Radarr copies → /mnt/media (NFS, Proxmox host)
         │  ← this is the ONE file copy. local disk → NFS.
         ▼  Webhook to Plex: "scan this path"
Plex reads /mnt/media (NFS, worker-media-01)
         │
         ▼
Available to stream
```

Hardlinks are not possible here (downloads = local ext4, media = NFS = different filesystems). One copy is correct and accepted — no second copy ever occurs. The separation of VMs is about resource isolation (Plex transcoding CPU vs Sonarr/Radarr DB I/O), not hardlink optimisation.

**NFS export must be:**
- Read-write for `worker-general-01` (Sonarr/Radarr write completed files)
- Read-only for `worker-media-01` (Plex only reads)

**Cross-VM library refresh:** Sonarr/Radarr and Plex are on different nodes and can't reach each other via Docker service DNS unless they share an overlay network. Both stacks must join a shared internal `media` overlay network. Plex exposes `32400` on that network; Sonarr/Radarr connect to `plex:32400` for webhook library refresh.

> [!note] Runbook patch issued this session — see `Runbook_Patch_Media_Split_2026-04-12.md`

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]
- [[Plex Worker Storage & Network Layout]]
