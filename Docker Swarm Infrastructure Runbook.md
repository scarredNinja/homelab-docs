---

project_id: Homelab-2025 
phase: "Phase 5: Docker Swarm" 
tags:

- Netbox
- DockerSwarm

---

# Docker Swarm Infrastructure Runbook

> [!abstract] Overview This runbook covers three sequential phases for migrating to a fully operational Docker Swarm homelab. **Complete phases in order** — Phase 2 depends on Phase 1 being stable.
> 
> A fourth section covers pre-migration storage setup and the Proxmox-only expansion path. Complete this before Phase 2 if you intend to add more Proxmox nodes or Raspberry Pis later.

**Tags:** #homelab #docker #swarm #proxmox #traefik #migration

---

## 📋 Quick Reference

### VM Layout

|VM|VLANs|vCPU / RAM|Labels|Services|
|---|---|---|---|---|
|`manager-01`|60|2 / 4GB|`role=manager`|Portainer|
|`traefik-dmz-01`|80 + 60|2 / 2GB|`zone=public`|Traefik only|
|`worker-monitoring-01`|60|2 / 4GB|`zone=monitoring`|Prometheus, Grafana, Loki, InfluxDB|
|`worker-media-01`|50|4 / 8GB|`type=media`|Plex, Tautulli|
|`worker-general-01`|40|4 / 6GB|`zone=mediamanagement` `nas=nfs`|Sonarr, Radarr, Prowlarr, Transmission+Gluetun|
|`worker-controller-01`|60 + 20|2 / 4GB|`zone=controller` `nas=nfs`|Home Assistant, UniFi|

> [!note] Media VM split (2026-04-12) `worker-media-01` runs Plex + Tautulli only (NFS read-only). `worker-general-01` runs the Arr stack + Transmission with a second local downloads disk and NFS read-write access. See [[#Media Data Flow]] below.

### Node Label Reference

|Label|Value|VM|Services|
|---|---|---|---|
|`node.role`|`manager`|manager-01|Portainer|
|`node.labels.zone`|`public`|traefik-dmz-01|Traefik|
|`node.labels.zone`|`monitoring`|worker-monitoring-01|Prometheus, Grafana, Loki, InfluxDB|
|`node.labels.type`|`media`|worker-media-01|Plex, Tautulli|
|`node.labels.zone`|`mediamanagement`|worker-general-01|Sonarr, Radarr, Prowlarr, Transmission|
|`node.labels.zone`|`controller`|worker-controller-01|Home Assistant, UniFi|
|`node.labels.nas`|`nfs`|general + controller|NFS-dependent services|

### Provision Commands

```bash
# traefik-dmz-01 — dual-homed: VLAN 60 + VLAN 80 DMZ
./02-provision-vm.sh --vlan 60 --nic2-vlan 80 --cores 2 --memory 2048 \
  --label zone=public traefik-dmz-01

# worker-monitoring-01 — VLAN 60, needs to reach all VLANs to scrape metrics
./02-provision-vm.sh --vlan 60 --cores 2 --memory 4096 \
  --label zone=monitoring worker-monitoring-01

# worker-media-01 — Plex + Tautulli only, NFS read-only for media
./02-provision-vm.sh --vlan 50 --cores 4 --memory 8192 \
  --label type=media worker-media-01

# worker-general-01 — Arr stack + Transmission, second disk for downloads
# NOTE: add a second disk (500GB+) for /mnt/downloads at provision time
./02-provision-vm.sh --vlan 40 --cores 4 --memory 6144 \
  --label zone=mediamanagement,nas=nfs worker-general-01

# worker-controller-01 — dual-homed: VLAN 60 + IoT VLAN 20
./02-provision-vm.sh --vlan 60 --nic2-vlan 20 --cores 2 --memory 4096 \
  --label zone=controller,nas=nfs worker-controller-01
```

### Media Data Flow

```
Transmission (worker-general-01)
    │  Downloads to /mnt/downloads  ← local disk, staging only
    ▼
Sonarr / Radarr (worker-general-01)
    │  Moves completed file → /mnt/media (NFS, read-write)
    │  ONE file copy: local disk → NFS. No second copy ever occurs.
    │  Hardlinks not possible (different filesystems) — this is accepted.
    ▼
NFS share on Proxmox host (rpool/media)
    │
    ▼  Sonarr/Radarr webhook → Plex scan new path
Plex (worker-media-01)
    │  Reads /mnt/media ← NFS, read-only
    ▼
Available to stream
```

> [!important] Sonarr/Radarr must mount both `/mnt/downloads` and `/mnt/media` in the same container so the import is a move (one copy), not copy-then-delete. NFS export must be **read-write** for `worker-general-01` and **read-only** for `worker-media-01`.

> [!tip] Cross-VM Plex webhook Both stacks must join a shared internal `media` overlay network. Sonarr/Radarr connect to `plex:32400` on that network to trigger library refresh. Without this, Swarm service DNS won't resolve across nodes.

---

## Phase 0 — Pre-Migration Storage Setup

> [!abstract] Do this before Phase 2 — with one exception Complete Steps 0.2–0.5 before running the ZFS send/receive migration. Setting up PBS and snapshot jobs after real data has landed on the new node means you have an unprotected window. Setting up NFS exports after services are running requires stopping them to remount — do it now and it's invisible.
> 
> **Step 0.1 is the exception** — it's a verification step that requires a live VM and a completed PBS backup to be meaningful. Run it after `manager-01` is deployed and PBS has completed its first job, before proceeding to Phase 2.

---

### Step 0.1 — Verify PBS Captures Docker Data

> [!note] Do this after the first VM is deployed and PBS has run at least one backup This is a verification step, not a setup step — it only makes sense once there is a live VM and a completed backup to inspect. Run it after `manager-01` is up and PBS has completed its first job.

> [!warning] Why this matters PBS backs up VM disk images. Your Docker data lives in ZFS datasets passed through to VMs via virtiofs — these are host-side mounts, **not inside the VM's virtual disk**. If you rely solely on PBS without verifying this, your actual service data (Plex library, HA config, InfluxDB, etc.) may not be protected.

```bash
# On Proxmox host — confirm dataset paths exist on the host side
zfs list -r rpool/docker-data
zfs list -r rpool/docker-tsdb
zfs list -r rpool/docker-db
zfs list -r rpool/docker-swarm

# Check what the last PBS backup actually captured
proxmox-backup-client list
```

If the datasets are host-side passthroughs and not inside a VM's zvol, PBS is not protecting them. Proceed to Step 0.2 to add an independent data-layer backup.

> [!tip] Old pool → backup + Transmission storage Once the Phase 3 migration is complete and the old PVE node's data is confirmed stable on the new node, the old pool will be repurposed: wiped and used as the ZFS send target for Step 0.2 and as Transmission's download storage. Do not wipe the old pool until services have been stable for at least 7 days post-migration.

- [x] First VM (`manager-01`) deployed and running ✅ 2026-04-08
- [ ] PBS has completed at least one backup job
- [ ] Verified PBS backup scope — confirmed what is and is not captured
- [ ] Old pool repurposed as backup target + Transmission storage (post Phase 3)

---

### Step 0.2 — Configure ZFS Snapshot + Send Job

Add a snapshot and send job as an independent data-layer backup, separate from PBS VM backups.

```bash
cat << 'EOF' > /usr/local/bin/zfs-docker-snapshot.sh
#!/bin/bash
SNAP="daily-$(date +%Y%m%d)"
DATASETS="docker-data docker-tsdb docker-db docker-swarm"
SRCPOOL="rpool"
DSTPOOL="oldpool"

for ds in $DATASETS; do
  zfs snapshot ${SRCPOOL}/${ds}@${SNAP}
  zfs send ${SRCPOOL}/${ds}@${SNAP} | zfs receive -F ${DSTPOOL}/${ds}
  echo "$(date): snapshot + send ${ds} complete"
done

zfs list -t snapshot -o name | grep "daily-" | \
  awk -v cutoff="daily-$(date -d '7 days ago' +%Y%m%d)" '$0 < cutoff' | \
  xargs -r zfs destroy
EOF

chmod +x /usr/local/bin/zfs-docker-snapshot.sh
echo "0 2 * * * root /usr/local/bin/zfs-docker-snapshot.sh >> /var/log/zfs-snapshot.log 2>&1" \
  >> /etc/cron.d/zfs-docker-snapshots
```

- [x] Snapshot script created and tested manually ✅ 2026-04-10
- [x] Cron job added ✅ 2026-04-10
- [x] Backup pool or target confirmed reachable ✅ 2026-04-10

---

### Step 0.3 — Set Up PBS on New PVE Node

```bash
# Proxmox UI: Datacenter → Backup → Add
# Schedule: daily 03:00, Mode: Snapshot
# VMs: manager-01, traefik-dmz-01, worker-media-01, worker-controller-01, worker-general-01
```

- [ ] PBS storage added to new PVE node
- [ ] Backup jobs created for all Swarm VMs
- [ ] Test backup run completed successfully
- [ ] Verified backup appears in PBS datastore

---

### Step 0.4 — Export ZFS Datasets as NFS

```bash
apt install nfs-kernel-server -y

cat << 'EOF' >> /etc/exports
/rpool/docker-data    10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash)
/rpool/docker-tsdb    10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash)
/rpool/docker-db      10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash)
/rpool/docker-swarm   10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash)
/rpool/media          10.0.50.0/24(rw,sync,no_subtree_check,no_root_squash)
/rpool/media          10.0.40.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

exportfs -ra
systemctl enable --now nfs-kernel-server
exportfs -v
```

> [!warning] NFS export permissions The media share has **different access requirements per VM**: `worker-general-01` (VLAN 40) needs read-write to import completed downloads; `worker-media-01` (VLAN 50) needs read-only. Restrict at the export level — do not rely on container-level `:ro` mounts as the only control. Tighten the `10.0.0.0/8` ranges above to the specific VLAN subnets once confirmed.

- [x] NFS server installed on Proxmox host ✅ 2026-04-10
- [x] `/etc/exports` configured with correct CIDRs ✅ 2026-04-10
- [x] `exportfs -v` shows all four datasets ✅ 2026-04-10
- [x] NFS mount tested from a VM ✅ 2026-04-10
- [ ] Media NFS export added with per-VLAN read-write / read-only split

---

### Step 0.5 — Placement Constraint Discipline

> [!tip] Use shared zone labels on placement constraints — never node hostnames. This costs nothing now but means adding nodes later is just labelling them. No stack file changes required.

```yaml
# Wrong — tied to one node forever
deploy:
  placement:
    constraints:
      - node.hostname == worker-controller-01

# Right — any node with this label can run it
deploy:
  placement:
    constraints:
      - node.labels.zone == controller
```

- [ ] Reviewed all stack files — no hostname constraints present

---

## Phase 1 — Traefik DMZ Migration ✅ 2026-04-10

> [!success] Phase 1 Complete Traefik is deployed on `traefik-dmz-01` in the DMZ. Portainer deployed in agent mode. Both accessible via internal entrypoint with wildcard TLS.

### Prerequisites

- [x] `manager-01` provisioned, Swarm initialised, Portainer deployed ✅ 2026-04-08
- [x] `traefik-dmz-01` provisioned and joined to Swarm ✅ 2026-04-08
- [x] Provisioning scripts updated — Bug 6 (Portainer redeploy), Bug 7 (bridge cleanup), `--label` flag, secondary NIC netplan, systemd virtiofs ordering ✅ 2026-04-08
- [x] `node.labels.zone=public` applied to `traefik-dmz-01` ✅ 2026-04-08
- [x] DMZ NIC (`enp6s19`, MAC `52:54:c9:67:3c:d0`) — DHCP reservation in pfSense VLAN80 ✅ 2026-04-08
- [x] pfSense WAN alias `Cloudflare_IPs` created, port 443 NAT+firewall rule restricted ✅ 2026-04-08
- [x] Cloudflare DNS A record points to pfSense WAN IP ✅ 2026-04-08
- [x] Cloudflare SSL mode set to Full (strict) ✅ 2026-04-08
- [x] `cf_api_token` Docker secret created ✅ 2026-04-10

---

### Step 1.1 — Traefik Stack Architecture

The final deployed stack lives at `proxmox-swarm/stacks/traefik/stack-traefik.yml` in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home).

**Key design decisions:**

- `traefik:latest` — Docker 29.4 rejects API 1.24 used by pinned older images
- `mode: host` ports — preserves real Cloudflare source IPs; `mode: ingress` rewrites them through the Swarm mesh, breaking `trustedIPs`
- `tecnativa/docker-socket-proxy` sidecar on manager — `traefik-dmz-01` is a worker node and cannot list Swarm services from its local socket; proxy runs on manager and exposes a read-only API over the `traefik-backend` overlay network
- `traefik-backend` internal overlay — connects socket proxy on manager to Traefik on DMZ worker
- `traefik-public` external overlay — services attach to this to be discovered by Traefik

**Entrypoints:**

- `:80` → redirects to websecure
    
- `:443` → `websecure` (external, Cloudflare trusted IPs configured)
    
- `:8443` → `internal` (management, VLAN 60 only)
    
- [x] Traefik stack deployed ✅ 2026-04-10
    
- [x] `docker-socket-proxy` sidecar running on manager ✅ 2026-04-10
    
- [x] `traefik-backend` overlay network created ✅ 2026-04-10
    
- [x] `traefik-public` overlay network created ✅ 2026-04-10
    

---

### Step 1.2 — Dynamic Config Consolidation

All infrastructure routes consolidated into `traefik/dynamic/infrastructure.yml` in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home).

- pihole, pfsense, proxmox, extreme switch, PDU, portainer routes merged
    
- Individual route stubs deleted
    
- All routes using `internal` entrypoint with `websecure` TLS
    
- `insecureSkipVerify: true` transport for self-signed internal services (pfsense, proxmox)
    
- [x] Dynamic config consolidated into `infrastructure.yml` ✅ 2026-04-08
    
- [x] Stale individual stubs removed ✅ 2026-04-08
    
- [x] All stacks reorganised under `proxmox-swarm/stacks/` ✅ 2026-04-08
    
- [x] `CLAUDE.md` created documenting repo structure and conventions ✅ 2026-04-08
    

---

### Step 1.3 — Deploy Traefik and Portainer

```bash
# On Proxmox host — copy config from repo to virtiofs share
bash /root/copy-traefik-config.sh

# On manager-01 — create CF token secret
docker secret create cf_api_token /mnt/docker-data/traefik/data/cf_api_token.txt

# Deploy Traefik — pinned to traefik-dmz-01 via zone=public label
docker stack deploy -c /mnt/docker-swarm/stacks/traefik/stack-traefik.yml traefik

# Verify placement — must show traefik-dmz-01
docker service ps traefik_traefik

# Deploy Portainer
docker stack deploy -c /mnt/docker-swarm/portainer/stack-portainer.yml portainer
```

- [x] Traefik stack deployed ✅ 2026-04-10
- [x] Service running on `traefik-dmz-01` ✅ 2026-04-10
- [x] Portainer deployed in agent mode ✅ 2026-04-10
- [x] Portainer agent running on all nodes ✅ 2026-04-10

---

### Step 1.4 — Verify

```bash
# Confirm Traefik is on correct node
docker service ps traefik_traefik
# Must show: traefik-dmz-01

# Confirm Portainer agent is on all nodes
docker service ps portainer_portainer_agent

# External test — from mobile data
curl -I https://yourdomain.com
# Expected: HTTP/2 200 with CF-Ray header
```

- [x] Traefik dashboard accessible at `https://traefik.home.purvishome.com:8443` ✅ 2026-04-10
- [x] Portainer accessible at `https://portainer.home.purvishome.com` ✅ 2026-04-10
- [x] Internal services routing correctly ✅ 2026-04-10
- [ ] External HTTPS test returns `CF-Ray` header
- [ ] DMZ isolation confirmed — `traefik-dmz-01` cannot reach VLAN 60 services directly

---

## Phase 1.5 — Monitoring VM Provisioning ✅ 2026-04-12

> [!success] Phase 1.5 Complete Monitoring stack deployed on `worker-monitoring-01`. Prometheus, Grafana, Loki, InfluxDB running. Global exporters (node-exporter, cAdvisor, Promtail) running on all nodes. Grafana data source configuration and dashboard import pending next session.

> [!info] Why a dedicated VM not `manager-01` Monitoring on the manager means a manager crash leaves you blind and without control simultaneously. A dedicated `worker-monitoring-01` on VLAN 60 keeps the observability plane independent. VLAN 60 is the correct placement — it has existing firewall access to all other infrastructure VLANs needed for scraping metrics.

### Cleanup Before Phase 1.5

The following stale references exist in older notes and should not be followed:

- `Docker_Swarm_Monitoring.md` references VLAN 85, 65, and 90 for monitoring — these VLANs are not in current topology. Monitoring lives on **VLAN 60**.
- `Monitoring_and_Responsibilities.md` references a standalone `swarm-monitoring` VM on VLAN 90 — superseded by this plan.
- IP table entries for Grafana at `10.0.70.10` and Prometheus at `10.0.60.11` are stale — these will be DHCP-assigned Swarm services, not static IPs.
- `stack.yaml`, `promethus.yml`, `prometheus + grafana + loki.yml` in Phase 8 spoke notes are old drafts — the stack file in this runbook supersedes them.

---

### Step 1.5.1 — Provision `worker-monitoring-01`

```bash
# On Proxmox host
./02-provision-vm.sh --vlan 60 --cores 2 --memory 4096 \
  --label zone=monitoring worker-monitoring-01

./03-post-boot.sh worker-monitoring-01

# Verify joined
docker node ls
# Should show worker-monitoring-01 as Ready
```

> [!bug] Bug #8 — Auto-join silent failure (2026-04-12) `worker-monitoring-01` did not auto-join via `03-post-boot.sh`. Join and label had to be applied manually. Root cause: script does not verify join success, and does not apply node labels on the manager post-join. Claude Code task raised to fix.

```bash
# Manual recovery if auto-join fails
# On the worker VM:
docker swarm join --token <worker-token> 10.0.60.30:2377

# On manager-01:
docker node update --label-add zone=monitoring \
  $(docker node ls --filter name=worker-monitoring-01 -q)
```

- [x] `worker-monitoring-01` provisioned and joined Swarm ✅ 2026-04-12
- [x] `zone=monitoring` label confirmed ✅ 2026-04-12

---

### Step 1.5.2 — Prepare Directories

Run on Proxmox host — these paths are on the virtiofs shares that `worker-monitoring-01` mounts automatically:

```bash
# Config and data (docker-data virtiofs)
mkdir -p /mnt/docker-data/prometheus
mkdir -p /mnt/docker-data/grafana/data
mkdir -p /mnt/docker-data/grafana/provisioning
mkdir -p /mnt/docker-data/loki/data
mkdir -p /mnt/docker-data/promtail

# TSDB on its own dataset — different recordsize tuning from config data
mkdir -p /mnt/docker-tsdb/prometheus
mkdir -p /mnt/docker-tsdb/influxdb

# Ownership — containers run as non-root UIDs
# Prometheus: UID 65534 (nobody), Grafana: UID 472, Loki: UID 10001
chown -R 65534:65534 /mnt/docker-tsdb/prometheus
chown -R 472:472 /mnt/docker-data/grafana/data
chown -R 10001:10001 /mnt/docker-data/loki/data
```

> [!warning] Always set ownership before deploying. Prometheus (UID 65534), Grafana (UID 472), and Loki (UID 10001) fail silently if the host directory is root-owned — they restart in a loop with no obvious error message.

- [x] Directories created on Proxmox host ✅ 2026-04-12
- [x] Ownership set correctly ✅ 2026-04-12

---

### Step 1.5.3 — Prometheus Config

File: `/mnt/docker-data/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    dns_sd_configs:
      - names: ['tasks.monitoring_node-exporter']
        type: 'A'
        port: 9100

  - job_name: cadvisor
    dns_sd_configs:
      - names: ['tasks.monitoring_cadvisor']
        type: 'A'
        port: 8080

  - job_name: traefik
    metrics_path: /metrics
    static_configs:
      - targets: ['traefik_traefik:8080']
```

> [!tip] `tasks.<stack>_<service>` DNS resolves to every running replica IP automatically. New nodes picked up within one scrape interval — no config changes needed.

> [!note] Phase 8 additions Add scrape blocks for Proxmox exporter, pfSense SNMP, and app exporters when Phase 8 targets are ready. See [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]].

- [x] `prometheus.yml` created ✅ 2026-04-12

InfluxDB secret: `docker secret create influx_pw /mnt/docker-swarm/configs/influx_password.txt`

---

### Step 1.5.4 — Promtail Config

File: `/mnt/docker-data/promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets: [localhost]
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag: attrs.tag
          source: attrs
      - regex:
          expression: (?P<container_name>(?:[^|]*))\|
          source: tag
      - timestamp:
          format: RFC3339Nano
          source: time
      - labels:
          stream:
          container_name:
      - output:
          source: output
```

- [x] `promtail-config.yml` created ✅ 2026-04-12

---

### Step 1.5.5 — Monitoring Stack File

Stack file: `proxmox-swarm/stacks/monitoring/stack-monitoring.yml` in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home).

**Key design decisions:**

- Prometheus, Grafana, Loki, InfluxDB pinned to `zone=monitoring` (`worker-monitoring-01`)
    
- node-exporter, cAdvisor, Promtail run as `mode: global` — one per node across the entire Swarm
    
- node-exporter uses `network_mode: host` — required for accurate host-level metrics; it does not join overlay networks
    
- TSDB (Prometheus, InfluxDB) on `/mnt/docker-tsdb` — separate virtiofs dataset with different recordsize tuning
    
- Grafana and Loki config/data on `/mnt/docker-data`
    
- Prometheus and Grafana on `traefik-public` network for dashboard routing; all services on internal `monitoring` overlay for scraping
    
- Global services have `delay: 10s` restart delay to prevent thundering-herd on Swarm rebuild
    
- [x] Stack deployed via Portainer ✅ 2026-04-12
    

---

### Step 1.5.6 — Deploy

```bash
# On manager-01
docker stack deploy -c /mnt/docker-swarm/stacks/monitoring/stack-monitoring.yml monitoring

# Watch rollout
watch docker stack ps monitoring

# Verify all services up
docker service ls | grep monitoring
```

- [x] Monitoring stack deployed ✅ 2026-04-12
- [x] Prometheus, Grafana, Loki, InfluxDB running on `worker-monitoring-01` ✅ 2026-04-12
- [x] node-exporter, cAdvisor, Promtail running on all nodes ✅ 2026-04-12
- [x] `https://prometheus.home.purvishome.com` loads ✅ 2026-04-12
- [x] `https://grafana.home.purvishome.com` loads ✅ 2026-04-12
- [x] Prometheus targets page: node-exporter, cAdvisor, Traefik all `UP` [priority:: 1] ✅ 2026-04-17

---

### Step 1.5.7 — Grafana Initial Setup

```
Login: admin / admin → change immediately
```

|Source|Type|URL|
|---|---|---|
|Prometheus|Prometheus|`http://prometheus:9090`|
|Loki|Loki|`http://loki:3100`|
|InfluxDB|InfluxDB (Flux)|`http://influxdb:8086` — org `home`, bucket `proxmox_metrics`|

Import dashboards:

|ID|Name|Source|
|---|---|---|
|1860|Node Exporter Full|Prometheus|
|14282|cAdvisor|Prometheus|
|609|Docker Swarm|Prometheus|
|13639|Loki logs|Loki|

- [x] Admin password changed ✅ 2026-04-12
- [x] Prometheus data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] Loki data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] InfluxDB data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] Core dashboards imported [priority:: 2] ✅ 2026-04-17
- [ ] look at other database migrations [priority:: 2]

---

### Step 1.5.8 — Hand off to Phase 8

> [!abstract] Core monitoring is now running. Phase 8 ([[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]) covers the full monitoring target build-out. Complete Phase 2 (remaining VMs) first so all scrape targets exist, then work through Phase 8.

Phase 8 adds: Proxmox host metrics (`pve-exporter`), pfSense metrics (SNMP exporter), Alertmanager, app-specific exporters (Plex, Sonarr, Radarr, Home Assistant, UniFi), Uptime Kuma for external service checks, dashboard refinement and alert rules.

---

## Phase 2 — Remaining VM Provisioning

> [!abstract] Deploy remaining worker VMs after monitoring is stable. Having Prometheus and Grafana running means you get instant visibility into each new VM as it joins.

### Step 2.1 — Provision Worker VMs

Run from Proxmox host. Provision one at a time and verify each joins cleanly before proceeding.

```bash
# worker-media-01 — Plex + Tautulli only
./02-provision-vm.sh --vlan 50 --cores 4 --memory 8192 \
  --label type=media worker-media-01
./03-post-boot.sh worker-media-01

docker node ls

# worker-general-01 — Arr stack + Transmission
# NOTE: provision with a second disk (500GB+) for downloads staging
./02-provision-vm.sh --vlan 40 --cores 4 --memory 6144 \
  --label zone=mediamanagement,nas=nfs worker-general-01
./03-post-boot.sh worker-general-01

docker node ls

# worker-controller-01 — dual-homed: VLAN 60 + IoT VLAN 20
./02-provision-vm.sh --vlan 60 --nic2-vlan 20 --cores 2 --memory 4096 \
  --label zone=controller,nas=nfs worker-controller-01
./03-post-boot.sh worker-controller-01
```

> [!note] Portainer stack auto-redeploys after each join `03-post-boot.sh` now handles this automatically — `redeploy_portainer_after_join()` runs after each worker joins so overlay networks propagate correctly.

> [!warning] Auto-join may fail silently (Bug #8) If a VM doesn't appear in `docker node ls`, SSH in and join manually. Apply the node label on `manager-01` afterwards. See Step 1.5.1 for manual recovery commands.

- [ ] Go over media and media management setup to cross-check - need to also pay attention to how migration will work
- [ ] `worker-media-01` provisioned and joined Swarm
- [ ] `worker-general-01` provisioned and joined Swarm, second disk mounted at `/mnt/downloads`
- [ ] `worker-controller-01` provisioned and joined Swarm
- [ ] All nodes show `Ready` in `docker node ls`
- [ ] node-exporter and cAdvisor appear in Prometheus for all new nodes

---

## Phase 3 — Legacy Service Migration

> [!abstract] Strategy Services run as Docker Compose on the old server (same Proxmox host, different ZFS pool). Migration uses **ZFS send/receive** — fast, atomic, and snapshot-consistent. No data crosses the network.

### Migration Overview

|Service|Old Data Path|Destination VM|Downtime|
|---|---|---|---|
|Plex|`/opt/plex/config`|`worker-media-01`|~5 min|
|Tautulli|`/opt/tautulli`|`worker-media-01`|~2 min|
|Sonarr|`/opt/sonarr/config`|`worker-general-01`|~2 min|
|Radarr|`/opt/radarr/config`|`worker-general-01`|~2 min|
|Transmission|`/opt/transmission`|`worker-general-01`|~2 min|
|Prowlarr|`/opt/prowlarr/config`|`worker-general-01`|~2 min|
|Home Asst.|`/opt/homeassistant`|`worker-controller-01`|~5 min|
|UniFi|`/opt/unifi`|`worker-controller-01`|~5 min|
|Grafana|`/opt/grafana`|`worker-monitoring-01`|~2 min|
|Prometheus|`/opt/prometheus/data`|`worker-monitoring-01`|~5 min|
|InfluxDB|`/opt/influxdb`|`worker-monitoring-01`|~5 min|

> [!tip] Priority Order Migrate in this order: **Plex → HA → UniFi → Sonarr/Radarr → InfluxDB → Grafana → Prometheus**. Prometheus history is acceptable to lose — it rebuilds within hours.

---

### Step 3.1 — Snapshot All Old Datasets

```bash
SNAP="migrate-$(date +%Y%m%d)"

for ds in plex tautulli sonarr radarr transmission prowlarr homeassistant unifi grafana prometheus influxdb; do
  zfs snapshot oldpool/docker/${ds}@${SNAP}
  echo "Snapshot: oldpool/docker/${ds}@${SNAP}"
done
```

- [ ] All snapshots created
- [ ] Verified with `zfs list -t snapshot | grep migrate`

---

### Step 3.2 — Stop Services on Old Server

```bash
cd /opt/compose
docker compose -f media.yml        down
docker compose -f controllers.yml  down
docker compose -f monitoring.yml   down
```

> [!warning] Home Assistant Shutdown Run `ha core stop` before stopping the HA container. HA uses SQLite — unclean shutdown can corrupt the database.

- [ ] Media services stopped
- [ ] Home Assistant stopped cleanly
- [ ] UniFi controller stopped
- [ ] Monitoring stack stopped
- [ ] All containers confirmed down: `docker ps` returns empty

---

### Step 3.3 — ZFS Send/Receive to New Pool

```bash
SNAP="migrate-$(date +%Y%m%d)"

# Plex + Tautulli → worker-media-01
for ds in plex tautulli; do
  zfs send oldpool/docker/${ds}@${SNAP} | zfs receive -F rpool/docker-data/${ds}
done

# Arr stack → worker-general-01
for ds in sonarr radarr transmission prowlarr; do
  zfs send oldpool/docker/${ds}@${SNAP} | zfs receive -F rpool/docker-data/${ds}
done

# Controllers → worker-controller-01
for ds in homeassistant unifi; do
  zfs send oldpool/docker/${ds}@${SNAP} | zfs receive -F rpool/docker-data/${ds}
done

# Monitoring → worker-monitoring-01
for ds in grafana prometheus influxdb; do
  zfs send oldpool/docker/${ds}@${SNAP} | zfs receive -F rpool/docker-data/${ds}
done
```

- [ ] Media datasets transferred
- [ ] Arr stack datasets transferred
- [ ] Controller datasets transferred
- [ ] Monitoring datasets transferred

---

### Step 3.4 — Verify Data on Target VMs

```bash
ssh ubuntu@worker-media-01
ls /mnt/docker-data/plex/config/Library
ls /mnt/docker-data/tautulli/config

ssh ubuntu@worker-general-01
ls /mnt/docker-data/sonarr/config
ls /mnt/docker-data/radarr/config
ls /mnt/docker-data/transmission/config

ssh ubuntu@worker-controller-01
ls /mnt/docker-data/homeassistant/configuration.yaml
ls /mnt/docker-data/unifi/data

ssh ubuntu@worker-monitoring-01
ls /mnt/docker-data/influxdb
ls /mnt/docker-data/grafana
```

- [ ] Plex library visible on `worker-media-01`
- [ ] Tautulli config visible on `worker-media-01`
- [ ] Sonarr/Radarr/Prowlarr/Transmission configs visible on `worker-general-01`
- [ ] HA `configuration.yaml` visible on `worker-controller-01`
- [ ] UniFi data visible on `worker-controller-01`
- [ ] InfluxDB and Grafana data visible on `worker-monitoring-01`

---

### Step 3.5 — Deploy Swarm Stacks

```bash
# Create shared media overlay network first — needed for Plex webhook from Arr stack
docker network create --driver overlay --attachable media

docker stack deploy -c /mnt/docker-swarm/stacks/plex/stack-plex.yml         plex
docker stack deploy -c /mnt/docker-swarm/stacks/arr/stack-arr.yml             arr
docker stack deploy -c /mnt/docker-swarm/stacks/controllers/stack-controllers.yml controllers

watch docker stack ps plex
watch docker stack ps arr
watch docker stack ps controllers
```

> [!important] `media` overlay network Must exist before deploying either stack. Plex and Sonarr/Radarr both join it so Sonarr/Radarr can reach `plex:32400` for library refresh webhooks across VM boundaries.

- [ ] `media` overlay network created
- [ ] Plex stack deployed and running on `worker-media-01`
- [ ] Arr stack deployed and running on `worker-general-01`
- [ ] Controllers stack deployed and running

---

### Step 3.6 — Post-Migration Validation

|Service|Check|Pass?|
|---|---|---|
|Plex|Library intact, no missing media|☐|
|Sonarr|System → Status: no config errors, series visible|☐|
|Radarr|System → Status: no config errors, movies visible|☐|
|Sonarr→Plex webhook|Add test item — Plex library updates automatically|☐|
|Home Asst.|States populated, integrations online|☐|
|UniFi|Devices adopted and connected|☐|
|InfluxDB|`influx ping` returns OK|☐|
|Grafana|Dashboards load, data sources connected|☐|
|Transmission|Web UI loads, torrent list intact|☐|
|Prowlarr|Indexers connected|☐|

---

> [!success] Phase 3 Complete All services migrated. Old server is now read-only. Do not delete old ZFS datasets for at least 7 days.

---

## Phase 4 — Service Deployment Reference

### Singleton vs Portable Services

> [!warning] Label Uniqueness For singleton services, the placement label **must be unique to one node**. If two nodes share `type=media`, Swarm may reschedule a service to the node where no data exists.

|Service|Singleton?|Constraint|Reason|
|---|---|---|---|
|Plex|✅ Hard pin|`type=media`|Config + transcode cache local; streams from NFS|
|Tautulli|✅ Hard pin|`type=media`|SQLite — must co-locate with Plex|
|Sonarr / Radarr|✅ Hard pin|`zone=mediamanagement`|SQLite; must see downloads disk + NFS simultaneously|
|Transmission|✅ Hard pin|`zone=mediamanagement`|Torrent resume state is local|
|Prowlarr|✅ Hard pin|`zone=mediamanagement`|SQLite, feeds Sonarr/Radarr on same node|
|Home Assistant|✅ Hard pin|`zone=controller`|Single-instance state machine, IoT VLAN binding|
|UniFi Controller|✅ Hard pin|`zone=controller`|MongoDB, device adoption tied to one controller|
|InfluxDB|✅ Hard pin|`zone=monitoring`|Local TSDB|
|Prometheus|✅ Hard pin|`zone=monitoring`|Local TSDB|
|Portainer|⚠️ Soft pin|`node.role==manager`|CE = one instance; agent runs everywhere|
|Traefik|✅ DMZ pin|`zone=public`|Single ingress|
|Grafana|✅ Pin|`zone=monitoring`|Dashboard state on virtiofs|

---

### Swarm Placement Constraints

```yaml
# Plex, Tautulli
deploy:
  placement:
    constraints:
      - node.labels.type == media

# Sonarr, Radarr, Prowlarr, Transmission
deploy:
  placement:
    constraints:
      - node.labels.zone == mediamanagement

# Traefik
deploy:
  placement:
    constraints:
      - node.labels.zone == public

# Controller services
deploy:
  placement:
    constraints:
      - node.labels.zone == controller

# Monitoring stack
deploy:
  placement:
    constraints:
      - node.labels.zone == monitoring
```

---

### Traefik Labels Reference

All stack files live in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home) under `proxmox-swarm/stacks/`.

> [!warning] Traefik v3 label change Use `traefik.swarm.network` not `traefik.docker.network` — the docker provider label is deprecated in Traefik v3.

```yaml
# Standard internal service
labels:
  - "traefik.enable=true"
  - "traefik.swarm.network=traefik-public"
  - "traefik.http.routers.<name>.rule=Host(`<name>.home.purvishome.com`)"
  - "traefik.http.routers.<name>.entrypoints=internal"
  - "traefik.http.routers.<name>.tls=true"
  - "traefik.http.routers.<name>.tls.certresolver=cloudflare"
  - "traefik.http.services.<name>.loadbalancer.server.port=<port>"

# External service (via websecure entrypoint)
labels:
  - "traefik.enable=true"
  - "traefik.swarm.network=traefik-public"
  - "traefik.http.routers.<name>.rule=Host(`<name>.purvishome.com`)"
  - "traefik.http.routers.<name>.entrypoints=websecure"
  - "traefik.http.routers.<name>.tls=true"
  - "traefik.http.routers.<name>.tls.certresolver=cloudflare"
  - "traefik.http.services.<name>.loadbalancer.server.port=<port>"
```

---

### Transmission + Gluetun VPN Sidecar

> [!note] Gluetun controls only Transmission's network namespace. All other services on `worker-general-01` use normal routing.

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun
    cap_add: [NET_ADMIN]
    environment:
      - VPN_SERVICE_PROVIDER=your_provider
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WG_PRIVATE_KEY}
    ports:
      - 9091:9091

  transmission:
    image: lscr.io/linuxserver/transmission
    network_mode: service:gluetun
    depends_on: [gluetun]
    volumes:
      - /mnt/docker-data/transmission/config:/config
      - /mnt/downloads:/downloads        # local disk — staging only
      - /mnt/media:/media                # NFS — final destination (read-write)
```

---

### Rollback Procedures

#### Phase 1 Rollback — Traefik

```bash
docker stack rm traefik
# Redeploy temporarily on manager (remove zone=public constraint)
# Update pfSense port 443 forward back to manager-01 IP
```

#### Phase 3 Rollback — Service Migration

```bash
# Restart old Compose services on old server
cd /opt/compose && docker compose -f media.yml up -d
```

> [!warning] Old ZFS snapshots are intact — data was only **copied**, never deleted. Do not remove old pool datasets until new services have been stable for **at least 7 days**.

---

## Appendix A — virtiofs Mount Reference

|Device|Tag|Mount Point|Purpose|
|---|---|---|---|
|`virtiofs0`|`docker-data`|`/mnt/docker-data`|Docker data root, service config|
|`virtiofs1`|`docker-tsdb`|`/mnt/docker-tsdb`|Time-series DB (Prometheus, InfluxDB)|
|`virtiofs2`|`docker-db`|`/mnt/docker-db`|Relational DB storage|
|`virtiofs3`|`docker-swarm`|`/mnt/docker-swarm`|Shared swarm configs, stack files|

```
/mnt/docker-data/<service>/config  → container /config
/mnt/docker-data/<service>/data    → container /data
/mnt/docker-tsdb/<service>         → TSDB data (Prometheus, InfluxDB)
/mnt/docker-swarm/stacks/<stack>/  → stack YAML files
```

---

## Appendix B — Proxmox Stage Expansion Reference

### Storage Architecture

virtiofs passthroughs are host-side mounts. A second Proxmox node cannot see ZFS datasets on node 1. Before adding any new node, NFS exports must be in place (Step 0.4).

### Adding Raspberry Pis as Swarm Workers

```bash
# Join Pi to Swarm (run on Pi)
docker swarm join --token <worker-token> 10.0.60.30:2377

# Label Pi (run on manager-01)
docker node update --label-add zone=pi-worker \
                   --label-add arch=arm64 \
                   pi-hostname
```

- Pis are ARM — verify `linux/arm64` image variants before deploying
- Run stateless services only on Pis at this stage
- Do not mount NFS from Proxmox on Pi-resident services

### Backup Layers

|Layer|Tool|Covers|Schedule|
|---|---|---|---|
|VM recovery|PBS|VM disk images, OS, Docker config|Daily|
|Data recovery|ZFS snapshot + send|All four docker datasets|Daily 02:00|
|Pre-migration safety|ZFS snapshot (manual)|Point-in-time before major change|On demand|

---

## Appendix C — Stack File Reference

All stack files live in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home) under `proxmox-swarm/stacks/`.

|Stack|File path|Pinned to|
|---|---|---|
|Traefik|`stacks/traefik/stack-traefik.yml`|`zone=public`|
|Portainer|`portainer/stack-portainer.yml`|`node.role==manager`|
|Monitoring|`stacks/monitoring/stack-monitoring.yml`|`zone=monitoring`|
|Plex|`stacks/plex/stack-plex.yml`|`type=media`|
|Arr|`stacks/arr/stack-arr.yml`|`zone=mediamanagement`|
|Controllers|`stacks/controllers/stack-controllers.yml`|`zone=controller`|

Copy from repo to virtiofs share before deploying:

```bash
# On Proxmox host — run after any repo update
bash /root/copy-traefik-config.sh
```

---

## Appendix D — Gotchas Reference

> [!abstract] Hard-won fixes. Each entry has the symptom, root cause, and fix. Entries marked `03-post-boot.sh` are now automated — manual steps only needed outside normal provisioning flow.

|#|Area|Symptom|Root Cause|Fix|
|---|---|---|---|---|
|1|cloud-init|`cicustom` vendor config silently ignored|Ubuntu 24.04 defaults to NoCloud datasource|Inject `99-pve.cfg` into image via `qemu-nbd` before Proxmox import|
|2|cloud-init|Setting `cicustom user=` separately drops `vendor=`|Proxmox bug|Always set `cicustom` as `user=...,vendor=...` in a single `qm set` call|
|3|Docker storage|Overlay2 driver fails on virtiofs mounts|virtiofs lacks `d_type` support|Use `fuse-overlayfs` storage driver in `/etc/docker/daemon.json`|
|4|Bash scripting|Script crashes when loop index hits 0|`(( idx++ ))` returns exit code 1, triggers `set -e`|Use `idx=$(( idx + 1 ))` instead|
|5|cloud-init|UTF-8 chars or CRLF in heredocs break YAML silently|Windows line endings or special chars|Strip with `sed 's/\r//'`; avoid non-ASCII in heredocs|
|6|ZFS|Mounting ZFS zvols via kpartx corrupts EXT4|ZFS and kpartx incompatible|Use `qemu-nbd` on source image instead|
|7|Docker networking|`dockerd` fails: `cannot create network ... networks have same bridge name`|Ghost bridge interfaces and stale `local-kv.db` survive Docker crash|Pre-flight bridge flush + `rm -rf /var/lib/docker/network/files/local-kv.db`. Baked into `03-post-boot.sh`.|
|8|Portainer|Agent only schedules on manager; workers reject scoped network|Overlay network not propagated to nodes that joined after initial stack deploy|Redeploy Portainer stack after all nodes joined. `03-post-boot.sh` handles this automatically.|
|9|SSH|`BatchMode=yes` fails with passphrase-protected key|Passphrase requires interactive input|Use passphrase-free SSH keys for automation|
|10|virtiofs|Containers can't access virtiofs-mounted directories|Incorrect ownership on host-side ZFS dataset|`chown -R <uid>:<gid> /path/to/dataset` on Proxmox host|
|11|Traefik|Container fails to start: `acme.json: permission denied`|`acme.json` must exist with `chmod 600` before Traefik starts|`touch acme.json && chmod 600 acme.json`|
|12|Provisioning|`CI_USER` env var mismatches across scripts|Default not persisted between shell sessions|Always `export CI_USER=ubuntu` explicitly|
|13|Proxmox API|`qm agent` JSON parsing fails|API returns plain array, not `{"result":[]}`|Parse raw array directly|
|14|DHCP|VM gets no IP after provisioning|VLAN tag not set on NIC|Set VLAN tag explicitly in `02-provision-vm.sh`|
|15|pfSense DynDNS|`Could not route to /client/v4/zones/token/dns_records`|pfSense CE 2.8.0 substitutes Username field literally into Cloudflare API URL|Use Zone ID as Username field. Get from Cloudflare Dashboard → domain → Overview → right sidebar.|
|16|Traefik v3 + Docker 29|Traefik rejects API 1.24|Docker 29.4 dropped support for old API versions|Use `traefik:latest` not a pinned older tag|
|17|Traefik on Swarm worker|Traefik can't list Swarm services|Worker nodes can't query Swarm API from local Docker socket|Deploy `tecnativa/docker-socket-proxy` on manager, connect via `traefik-backend` overlay. Set endpoint to `tcp://docker-proxy:2375`.|
|18|Traefik entrypoint name|`EntryPoint doesn't exist: https` in dynamic config|Entrypoint renamed from `https` to `websecure` in updated config|Update all dynamic config files: `entrypoints: [websecure]` not `[https]`|
|19|virtiofs + Docker|Swarm manager demotes itself to worker on reboot|Docker creates data subdirs with `700` on host-side ZFS mountpoint. virtiofs exposes host permissions directly — no remapping. Docker can't read Raft state DB on restart.|`chmod -R 755 /mnt/docker-data/` on Proxmox host after first Docker start. Must be host-level — VM-side fixes don't survive Docker restart. Automated in `03-post-boot.sh`.|
|20|virtiofs ownership|Containers fail silently on virtiofs-mounted paths|virtiofs exposes host-side ownership directly. Directories created by a previous user are inaccessible.|`chown -R root:root` on affected paths on Proxmox host. Verify with `ls -la /mnt/docker-swarm/`.|
|21|Swarm state DB|`Swarm: pending` / `Is Manager: false` after reboot or server switch|`docker-state.json` persists on virtiofs share with empty `AdvertiseAddr` — stale join-in-progress state from interrupted init survives VM rebuild|Clear `/mnt/docker-data/swarm/*` and `/mnt/docker-data/network/files/local-kv.db` on Proxmox host before re-init. `chmod -R 755 /mnt/docker-data/` after. Automated in `03-post-boot.sh`. See Appendix E for manual recovery.|
|22|Traefik internal routes|Internal services reachable from DMZ|DMZ VLAN 80 included in `internal-only` middleware IP allowlist|Remove VLAN 80 from internal allowlist. Only LAN + VLANs 20/40/50/60 should be in the allowlist.|
|23|Provisioning auto-join|Worker VM provisioned but does not join Swarm|`03-post-boot.sh` does not verify join success and does not apply node labels on manager post-join|Manual recovery: join on worker, apply label on manager. Claude Code task raised to fix script. See Step 1.5.1.|

---

## Appendix E — Swarm Recovery Procedure

> [!warning] Use this when `manager-01` reports `Swarm: pending` or `Is Manager: false` after reboot. This re-initialises the Swarm from scratch — all worker nodes must re-join afterwards. Under normal provisioning, `03-post-boot.sh` prevents this from occurring.

### Step 1 — Diagnose (manager-01)

```bash
sudo docker info | grep -A5 "Swarm"
sudo journalctl -u docker --since "10 minutes ago" --no-pager | grep -i "swarm\|raft\|error" | head -30
```

|State|Meaning|Action|
|---|---|---|
|`Swarm: pending` + `Is Manager: false`|Stale Raft state — Docker can see state exists but can't read/acquire it|Proceed to Step 2|
|`Swarm: inactive`|Docker has no Swarm state at all|Skip to Step 4 — run `swarm init` directly|
|`Swarm: active` + `Is Manager: true`|Healthy — no action needed|—|

### Step 2 — Read state file (Proxmox host)

```bash
cat /mnt/docker-data/swarm/docker-state.json
```

Healthy `docker-state.json` has a populated `AdvertiseAddr` field. An empty `AdvertiseAddr` means the state is an incomplete join-in-progress — it must be cleared.

```json
// Corrupt — empty AdvertiseAddr
{"LocalAddr":"","RemoteAddr":"10.0.60.30:2377","AdvertiseAddr":"","JoinInProgress":false}

// Healthy — AdvertiseAddr populated
{"LocalAddr":"10.0.60.30:2377","RemoteAddr":"","AdvertiseAddr":"10.0.60.30:2377","JoinInProgress":false}
```

### Step 3 — Clear stale state (Proxmox host)

```bash
cp -r /mnt/docker-data/swarm /mnt/docker-data/swarm.bak.$(date +%Y%m%d)
rm -rf /mnt/docker-data/swarm/*
rm -rf /mnt/docker-data/network/files/local-kv.db 2>/dev/null || true
chmod -R 755 /mnt/docker-data/
```

### Step 4 — Re-initialise Swarm (manager-01)

```bash
sudo systemctl restart docker
sudo docker swarm init --advertise-addr 10.0.60.30
sudo docker info | grep -A3 "Swarm"
sudo docker node ls
```

### Step 5 — Re-join workers

```bash
# Get fresh token (manager-01)
sudo docker swarm join-token worker

# On each worker — leave old swarm and rejoin
sudo docker swarm leave --force
# Paste join command from above

# Re-apply labels (manager-01) — labels are wiped with Swarm rebuild
docker node update --label-add zone=public \
  $(docker node ls --filter name=traefik-dmz-01 -q)
docker node update --label-add zone=monitoring \
  $(docker node ls --filter name=worker-monitoring-01 -q)
docker node update --label-add type=media \
  $(docker node ls --filter name=worker-media-01 -q)
docker node update --label-add zone=mediamanagement \
  $(docker node ls --filter name=worker-general-01 -q)
docker node update --label-add zone=controller \
  $(docker node ls --filter name=worker-controller-01 -q)

sudo docker node ls
```

### Step 6 — Redeploy stacks (in order)

```bash
# 1. Portainer first
docker stack deploy -c /mnt/docker-swarm/portainer/stack-portainer.yml portainer

# 2. Traefik second — requires traefik-public network
docker stack deploy -c /mnt/docker-swarm/stacks/traefik/stack-traefik.yml traefik

# 3. Monitoring
docker stack deploy -c /mnt/docker-swarm/stacks/monitoring/stack-monitoring.yml monitoring

# 4. Application stacks last
docker network create --driver overlay --attachable media  # if not already present
docker stack deploy -c /mnt/docker-swarm/stacks/plex/stack-plex.yml plex
docker stack deploy -c /mnt/docker-swarm/stacks/arr/stack-arr.yml arr
docker stack deploy -c /mnt/docker-swarm/stacks/controllers/stack-controllers.yml controllers
```

> [!note] Deploy order matters Portainer must come before Traefik (agent network needs to exist). Traefik must come before application stacks (services need `traefik-public` network to attach to). The `media` overlay must exist before deploying either `plex` or `arr` stacks.