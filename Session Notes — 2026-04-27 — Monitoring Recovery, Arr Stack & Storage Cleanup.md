---
date: '2026-04-27T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
session_type: 'debugging, configuration, deployment'
status: Completed
tags:
  - DockerSwarm
  - InfluxDB
  - Grafana
  - Traefik
  - Monitoring
  - Transmission
  - Arr
  - Storage
  - ZFS
---

# Session Notes — 2026-04-27 — Monitoring Recovery, Arr Stack & Storage Cleanup

> [!abstract] Overview
> Continuation of 2026-04-26 session. Completed downloads migration, fixed InfluxDB auth post-migration, configured Grafana datasources (InfluxQL + Flux), fixed Traefik metrics endpoint, deployed arr stack and Transmission/Gluetun, cleared stale MainStorage VM disks, set downloads quota and backups dataset.

---

## Session Goal

Restore monitoring stack (InfluxDB, Grafana, Prometheus/Traefik metrics), complete downloads migration, deploy arr stack and Transmission, clean up MainStorage.

---

## Completed

### 1. Downloads Migration — Completed

Resumed rsync from `subvol-110` after `mv` failed cross-dataset:

```bash
rsync -av --remove-source-files \
  /MainStorage/subvol-110-disk-0/home/downloads/complete/ \
  /MainStorage/downloads/complete/
```

Confirmed complete — file counts matched. Destroyed old LXC subvolume:

```bash
zfs destroy MainStorage/subvol-110-disk-0
```

> [!note] `mv` across ZFS datasets is not atomic even on the same pool — they are separate filesystems. Always use `rsync --remove-source-files` for cross-dataset moves.

### 2. InfluxDB — Authorization Token Fix

Post-migration `authorization not found` errors flooding logs. Root cause: Proxmox was using a stale/old token after migration.

- Confirmed InfluxDB healthy: `influx ping` returned OK
- Org `home` and bucket `proxmox_metrics` survived migration intact
- Existing token confirmed valid: `scarredNinja's Token`
- Updated Proxmox InfluxDB output token to match

Reset InfluxDB password via CLI:

```bash
influx user password \
  --host http://localhost:8086 \
  --name scarredNinja \
  --token '<token>'
```

### 3. InfluxDB — v1 Compatibility Setup

Dashboard (Grafana ID 10048) uses InfluxQL — v1 compatibility required.

Created DBRP mapping:

```bash
influx v1 dbrp create \
  --host http://localhost:8086 \
  --token '<token>' \
  --db proxmox_metrics \
  --rp autogen \
  --bucket-id 2296f78ab76a46c0 \
  --default
```

Created v1 auth for Grafana:

```bash
influx v1 auth create \
  --host http://localhost:8086 \
  --token '<token>' \
  --username grafana \
  --password <password> \
  --read-bucket 2296f78ab76a46c0 \
  --write-bucket 2296f78ab76a46c0
```

### 4. Grafana Datasources Configured

Two datasources set up:

| Name | Query Language | Use |
|------|---------------|-----|
| InfluxDB-InfluxQL | InfluxQL | Existing Proxmox dashboard (10048) |
| InfluxDB-Flux | Flux | New dashboard 15356 |

**InfluxQL datasource:**
- URL: `http://influxdb:8086`
- Basic auth: username `grafana`
- HTTP Header: `Authorization` → `Token <token>`
- Database: `proxmox_metrics`

**Flux datasource:**
- URL: `http://influxdb:8086`
- Org: `home`
- Token: `<token>`
- Default bucket: `proxmox_metrics`

Prometheus datasource updated to `http://prometheus:9090`.

### 5. Monitoring Stack — InfluxDB Network Fix

InfluxDB was only on the `monitoring` overlay (internal: true) — not on `traefik-public`. Traefik couldn't reach the backend despite correct labels.

Fixed in `stack-monitoring.yml`:

```yaml
influxdb:
  networks:
    - monitoring
    - traefik-public    # added
```

Also removed `DOCKER_INFLUXDB_INIT_*` environment variables — setup already complete, leaving them risks re-init attempts on restart.

> [!bug] See Gotcha #43

### 6. Traefik Metrics Endpoint Fix

Prometheus scraping `tasks.traefik_traefik:8080/metrics` returning 404.

**Root causes found and fixed:**

1. `insecure: false` → changed to `insecure: true` to re-enable port 8080
2. `metrics.prometheus` missing `entryPoint: traefik` — without this Traefik v3 doesn't serve metrics on the API port
3. Rogue config file overriding the correct config — deleted

**Final working config:**

```yaml
api:
  dashboard: true
  insecure: true

metrics:
  prometheus:
    entryPoint: traefik
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0
```

Prometheus scrape config confirmed correct:

```yaml
- job_name: 'traefik'
  dns_sd_configs:
    - names:
      - 'tasks.traefik_traefik'
      type: 'A'
      port: 8080
  metrics_path: /metrics
```

> [!bug] See Gotcha #44

### 7. Home Assistant Logs Reviewed

HA started successfully after migration. Notes:
- DB schema upgraded 43→53 on first start — expected
- Analytics connection refused — HA container can't reach internet (monitoring network is `internal: true`) — minor, non-blocking
- `redback` custom integration has two bugs (NoneType voltage calc, missing key) — needs update
- Otherwise healthy

### 8. Transmission + Gluetun Deployed

Started via `compose-vpn.yml` on `worker-mediamanagement-01`:

```bash
docker compose -f compose-vpn.yml up -d
```

Confirmed:
- Gluetun healthy (NordVPN connected)
- Transmission running, `/downloads/complete` visible inside container
- Volume mapping confirmed: `/mnt/downloads:/downloads`

### 9. Arr Stack Deployed

```bash
docker stack deploy -c /mnt/docker-swarm/stacks/stack-arr.yml arr
```

All services running 1/1 on `worker-mediamanagement-01`:

| Service | Status |
|---------|--------|
| `arr_sonarr` | Running ✅ |
| `arr_radarr` | Running ✅ |
| `arr_prowlarr` | Running ✅ |

Prowlarr configured in Sonarr and Radarr. Transmission download client added pointing to `gluetun:9091`. Import of existing downloads in progress.

### 10. MainStorage Cleanup — Stale VM Disks

Confirmed no active VMs referencing these datasets then destroyed:

```bash
zfs destroy MainStorage/vm-100-disk-0   # 20G
zfs destroy MainStorage/vm-106-disk-0   # 155G
zfs destroy MainStorage/vm-108-disk-0   # 33G
```

Total recovered: ~208G. MainStorage free: 608G → after backups reservation: 486G.

### 11. MainStorage — Downloads Quota + Backups Dataset

```bash
# Cap downloads
zfs set quota=500G MainStorage/downloads

# Create backups dataset
zfs create MainStorage/backups
zfs set compression=off MainStorage/backups
zfs set atime=off MainStorage/backups
zfs set reservation=150G MainStorage/backups
```

**Final MainStorage layout:**

| Dataset | Used | Available |
|---------|------|-----------|
| `MainStorage/downloads` | 442G | 58G (quota: 500G) |
| `MainStorage/backups` | 96K | 636G (reservation: 150G) |
| Pool free | — | 486G |

---

## Current State

| Service | Status |
|---------|--------|
| InfluxDB | ✅ Running, token fixed, v1 compat configured |
| Grafana | ✅ Both datasources working |
| Prometheus | ✅ Scraping Traefik metrics |
| Traefik | ✅ Metrics endpoint working |
| Home Assistant | ✅ Running (minor integration warnings) |
| Transmission + Gluetun | ✅ Running |
| Sonarr | ✅ Running, imports in progress |
| Radarr | ✅ Running, imports in progress |
| Prowlarr | ✅ Running, configured |
| MainStorage/downloads | ✅ 442G, quota set at 500G |
| MainStorage/backups | ✅ Created, 150G reserved |

---

## Next Session Priorities

1. **Verify arr imports complete** — check Sonarr/Radarr have matched all files from `/mnt/downloads/complete`
2. **Traefik entrypoint NIC binding** (deferred) — bind `websecure` → VLAN 80, `internal` → VLAN 60:443, fix `60-nic2-dmz.yaml` netplan permanently
3. **Plex stack** — deploy and verify media library
4. **Backup strategy** — configure ZFS send to `MainStorage/backups`
5. **Obsidian Git plugin** — set up auto-commit to `scarredNinja/homelab-docs`
6. **HA network** — fix outbound internet access for Home Assistant container
7. **Redback integration** — update custom component to fix NoneType bug

---

## Bugs Found This Session

> [!bug] Bug #43 — InfluxDB missing from traefik-public network
> **Symptom:** `influxdb.home.purvishome.com` unreachable via Traefik despite correct labels
> **Cause:** InfluxDB only on `monitoring` overlay (internal: true) — Traefik cannot reach backends on internal-only networks
> **Fix:** Add `traefik-public` to InfluxDB service networks in stack file. Any service needing Traefik routing must be on `traefik-public`.

> [!bug] Bug #44 — Traefik v3 metrics 404 on /metrics
> **Symptom:** Prometheus scraping `tasks.traefik_traefik:8080/metrics` returns 404
> **Causes (three):**
> 1. `api.insecure: false` disables port 8080 entirely
> 2. `metrics.prometheus` missing `entryPoint: traefik` — v3 requires explicit entrypoint declaration
> 3. Rogue config file in dynamic directory overriding static config
> **Fix:** Set `api.insecure: true`, add `entryPoint: traefik` to metrics config, audit dynamic config directory for stale files

> [!warning] DOCKER_INFLUXDB_INIT_MODE: setup in stack file
> Leaving init environment variables in the stack file after first setup is risky — on a clean data volume it would re-initialise and wipe existing data. Remove all `DOCKER_INFLUXDB_INIT_*` vars once setup is confirmed complete.

---

## Related Notes

- [[Session Notes — 2026-04-26 — rpool Full Recovery & Downloads Migration]]
- [[Docker Swarm Infrastructure Runbook]]
- [[Monitoring Docker Swarm]]
- [[Traefik]]
- [[InfluxDb]]
