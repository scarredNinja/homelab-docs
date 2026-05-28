---
type: reference
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
tags:
  - DockerSwarm
  - Runbook
  - Infrastructure
last_updated: 2026-04-21
---

# Docker Swarm Infrastructure Runbook

> [!abstract] Overview This runbook covers three sequential phases for migrating to a fully operational Docker Swarm homelab. **Complete phases in order** — Phase 2 depends on Phase 1 being stable.
> 
> A fourth section covers pre-migration storage setup and the Proxmox-only expansion path. Complete this before Phase 2 if you intend to add more Proxmox nodes or Raspberry Pis later.

**Tags:** #homelab #docker #swarm #proxmox #traefik #migration

---

## 📋 Quick Reference

### VM Layout

| VM                          | VLANs    | vCPU / RAM | Labels                           | Services                                       |
| --------------------------- | -------- | ---------- | -------------------------------- | ---------------------------------------------- |
| `manager-01`                | 60       | 2 / 4GB    | `role=manager`                   | Portainer                                      |
| `traefik-dmz-01`            | 80 + 60  | 2 / 2GB    | `zone=public`                    | Traefik only                                   |
| `worker-monitoring-01`      | 60       | 2 / 4GB    | `zone=monitoring`                | Prometheus, Grafana, Loki, InfluxDB            |
| `worker-media-01`           | 50 + 100 | 4 / 8GB    | `type=media`                     | Plex, Tautulli                                 |
| `worker-mediamanagement-01` | 50 + 100 | 6 / 6GB    | `zone=mediamanagement` `nas=nfs` | Sonarr, Radarr, Prowlarr, Transmission+Gluetun |
| `worker-controller-01`      | 60 + 20  | 2 / 4GB    | `zone=controller` `nas=nfs`      | Home Assistant, UniFi                          |
| `dev-node-01`               | 40       | 2 / 4GB    | `zone=dev` `env=development`     | Obsidian MCP, Syncthing                        |

> [!note] Media VM split (2026-04-12) `worker-media-01` runs Plex + Tautulli only (NFS read-only). `worker-mediamanagement-01` runs the Arr stack + Transmission with a second local downloads disk and NFS read-write access. See [[#Media Data Flow]] below.

> [!warning] `worker-general-01` removed 2026-05-24 — never provisioned. Media management role taken by `worker-mediamanagement-01`. Dev/homelab role replaced by `dev-node-01`.

### Node Label Reference

| Label              | Value             | VM                        | Services                                       |
| ------------------ | ----------------- | ------------------------- | ---------------------------------------------- |
| `node.role`        | `manager`         | manager-01                | Portainer                                      |
| `node.labels.zone` | `public`          | traefik-dmz-01            | Traefik                                        |
| `node.labels.zone` | `monitoring`      | worker-monitoring-01      | Prometheus, Grafana, Loki, InfluxDB            |
| `node.labels.type` | `media`           | worker-media-01           | Plex, Tautulli                                 |
| `node.labels.zone` | `mediamanagement` | worker-mediamanagement-01 | Sonarr, Radarr, Prowlarr, Transmission+Gluetun |
| `node.labels.zone` | `controller`      | worker-controller-01      | Home Assistant, UniFi                          |
| `node.labels.zone` | `dev`             | dev-node-01               | Dev app deployments                            |
| `node.labels.env`  | `development`     | all VLAN 40 nodes         | Environment marker                             |

### Provision Commands

```bash
# traefik-dmz-01 — dual-homed: VLAN 60 + VLAN 80 DMZ
./scripts/02-provision-vm.sh --vlan 60 --nic2-vlan 80 --cores 2 --memory 2048 \
  --label zone=public traefik-dmz-01

# worker-monitoring-01 — VLAN 60, scrapes all VLANs
./scripts/02-provision-vm.sh --vlan 60 --cores 2 --memory 4096 \
  --label zone=monitoring worker-monitoring-01

# worker-media-01 — dual-homed: VLAN 50 + VLAN 100 (NFS)
./scripts/02-provision-vm.sh --vlan 50 --nic2-vlan 100 \
  --cores 4 --memory 8192 --disk-size 40 \
  --disk2-size 100 --disk2-mount /mnt/transcode \
  --label type=media worker-media-01
./scripts/03-post-boot.sh worker-media-01 --nfs-mode ro

# worker-mediamanagement-01 — dual-homed: VLAN 50 + VLAN 100 (NFS)
./scripts/02-provision-vm.sh --vlan 50 --nic2-vlan 100 \
  --cores 4 --memory 6144 --disk-size 40 \
  --disk2-size 500 --disk2-mount /mnt/downloads \
  --label zone=mediamanagement worker-mediamanagement-01
./scripts/03-post-boot.sh worker-mediamanagement-01 --nfs-mode rw

# worker-controller-01 — dual-homed: VLAN 60 + IoT VLAN 20
./scripts/02-provision-vm.sh --vlan 60 --nic2-vlan 20 --cores 2 --memory 4096 \
  --label zone=controller worker-controller-01

# dev-node-01 — VLAN 40, dev workloads (Obsidian MCP, Syncthing)
./scripts/02-provision-vm.sh --vlan 40 --cores 2 --memory 4096 --disk-size 40 \
  --label zone=dev,env=development dev-node-01
./scripts/03-post-boot.sh dev-node-01
```

### Media Data Flow

```
Transmission (worker-mediamanagement-01)
    │  Downloads to /mnt/downloads  ← local disk, staging only
    ▼
Sonarr / Radarr (worker-mediamanagement-01)
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

> [!important] Sonarr/Radarr must mount both `/mnt/downloads` and `/mnt/media` in the same container so the import is a move (one copy), not copy-then-delete. NFS export must be **read-write** for `worker-mediamanagement-01` and **read-only** for `worker-media-01`.

> [!tip] Cross-VM Plex webhook Both stacks must join a shared internal `media` overlay network. Sonarr/Radarr connect to `plex:32400` on that network to trigger library refresh. Without this, Swarm service DNS won't resolve across nodes.

---

## 🎯 Current Priorities

> [!note] Updated at the start of each session. Single source of truth for
> active work. Backlog items live in session notes — promote here when active.

| # | Task | Status | Notes |
|---|------|--------|-------|
| 1 | Reboot `worker-media-01` — verify `--cpu host` took effect, retest Plex transcode | ⏳ Pending | Set this session, reboot not yet done |
| 2 | Verify overnight backup cron — all 4 logs + Grafana dashboard | ⏳ Pending | Check after 05:15 NZST |
| 3 | Plex — H264 + AAC Direct Play retest after VM reboot | 🔲 Pending | Expect Direct Play, no transcode |
| 4 | `traefik-dmz-01` — permanent `use-routes: false` on `enp6s19` netplan | ⏳ Pending | Temp fix 2026-04-21, does not survive reboot. See Appendix F |
| 5 | Plex — GPU for hardware transcoding (NVIDIA T400 low-profile) | 🔲 Backlog | PCIe passthrough to worker-media-01 |
| 6 | Obsidian Git plugin — auto-commit vault to `scarredNinja/homelab-docs` | 🔲 Backlog | |
| 7 | Homepage dashboard | ✅ Done | `stack-homepage.yml` deployed via PR #23 2026-05-22; hostname `dashboard.home.purvishome.com` |

**Status key:** ✅ Done · ⏳ Pending · 🔄 In Progress · 🔲 Backlog

---

## Phase 0 — Pre-Migration Storage Setup ✅ 2026-05-08

> [!abstract] Do this before Phase 2 — with one exception Complete Steps 0.2–0.5 before running the ZFS send/receive migration. Setting up PBS and snapshot jobs after real data has landed on the new node means you have an unprotected window. Setting up NFS exports after services are running requires stopping them to remount — do it now and it's invisible.
> 
> **Step 0.1 is the exception** — it's a verification step that requires a live VM and a completed PBS backup to be meaningful. Run it after `manager-01` is deployed and PBS has completed its first job, before proceeding to Phase 2.

---

### Step 0.1 — Verify PBS Captures Docker Data - ❌ Decided Against — 2026-05-07

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
- [x] PBS has completed at least one backup job — N/A, decided against ❌ 2026-05-07
- [x] Verified PBS backup scope — N/A, decided against ❌ 2026-05-07
- [x] Old pool repurposed as backup target + Transmission storage (post Phase 3) [priority:: 6] #backlog

---

### Step 0.2 — Configure ZFS Snapshot + Send Job  ✅ 2026-04-10

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

### Step 0.3 — Set Up PBS on New PVE Node ❌ Decided Against — 2026-05-07

> [!note] PBS evaluated and decided against (2026-05-07)
> Current backup chain provides sufficient coverage: ZFS send → MainStorage (daily), vzdump VM backups → MainStorage (nightly), restic off-site → Synology (nightly). PBS adds operational overhead without meaningful benefit given these layers. Items below closed.

- [x] PBS storage added to new PVE node — N/A, decided against ❌ 2026-05-07
- [x] Backup jobs created for all Swarm VMs — covered by `vm-backup.sh` (vzdump) ✅ 2026-05-07
- [x] Test backup run completed successfully — N/A, decided against ❌ 2026-05-07
- [x] Verified backup appears in PBS datastore — N/A, decided against ❌ 2026-05-07

---

### Step 0.4 — Export ZFS Datasets as NFS ✅ 2026-04-18

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

> [!warning] NFS export permissions The media share has **different access requirements per VM**: `worker-mediamanagement-01` (VLAN 50) needs read-write to import completed downloads; `worker-media-01` (VLAN 50) needs read-only. Restrict at the export level — do not rely on container-level `:ro` mounts as the only control. Tighten the `10.0.0.0/8` ranges above to the specific VLAN subnets once confirmed.

- [x] NFS server installed on Proxmox host ✅ 2026-04-10
- [x] `/etc/exports` configured with correct CIDRs ✅ 2026-04-10
- [x] `exportfs -v` shows all four datasets ✅ 2026-04-10
- [x] NFS mount tested from a VM ✅ 2026-04-10
- [x] Media NFS export added with per-VLAN read-write / read-only split ✅ 2026-04-18 (`worker-media-01` RO, `worker-mediamanagement-01` RW via VLAN 100)

---

### Step 0.5 — Placement Constraint Discipline ✅ 2026-04-19

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

- [x] Reviewed all stack files — no hostname constraints present ✅ 2026-04-19

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

### Step 1.1 — Traefik Stack Architecture ✅ 2026-04-10

The final deployed stack lives at `proxmox-swarm/stacks/traefik/stack-traefik.yml` in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home).

**Key design decisions:**

- `traefik:latest` — Docker 29.4 rejects API 1.24 used by pinned older images. **As of PR #61 (2026-05-26) all stack images are pinned to specific stable versions**; the original rationale for keeping Traefik unpinned no longer applies.
- `mode: host` ports — preserves real Cloudflare source IPs; `mode: ingress` rewrites them through the Swarm mesh, breaking `trustedIPs`
- `tecnativa/docker-socket-proxy` sidecar on manager — `traefik-dmz-01` is a worker node and cannot list Swarm services from its local socket; proxy runs on manager and exposes a read-only API over the `traefik-backend` overlay network
- `traefik-backend` internal overlay — connects socket proxy on manager to Traefik on DMZ worker
- `traefik-public` external overlay — services attach to this to be discovered by Traefik

> [!important] Overlay subnets are pinned — use `10.200.x.0/24` for all new overlays
> Docker auto-assigns overlay subnets from `10.0.0.0/8`, which collides with physical VLANs (Bug #32, 2026-04-21). All stacks redeployed with pinned IPAM subnets. **All future overlay networks must use `10.200.x.0/24`.**
> - `traefik-public` → `10.200.2.0/24`
> - `traefik_traefik-backend` → `10.200.3.0/24`

**Entrypoints:**

- `:80` → redirects to websecure
    
- `:443` → `websecure` (external, Cloudflare trusted IPs configured)
    
- `:8443` → `internal` (management, VLAN 60 only)
    
- [x] Traefik stack deployed ✅ 2026-04-10
    
- [x] `docker-socket-proxy` sidecar running on manager ✅ 2026-04-10
    
- [x] `traefik-backend` overlay network created ✅ 2026-04-10
    
- [x] `traefik-public` overlay network created ✅ 2026-04-10
    

---

### Step 1.2 — Dynamic Config Consolidation ✅ 2026-04-08

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

### Step 1.3 — Deploy Traefik and Portainer ✅ 2026-04-10

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
- [x] External HTTPS test returns `CF-Ray` header ✅ 2026-04-23 (via Cloudflare Tunnel — CGNAT blocks direct port forward, see Gotcha #33)
- [ ] DMZ isolation confirmed — `traefik-dmz-01` cannot reach VLAN 60 services directly [priority:: 6] #backlog

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

### Step 1.5.1 — Provision `worker-monitoring-01` ✅ 2026-04-12

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

### Step 1.5.2 — Prepare Directories ✅ 2026-04-12

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

### Step 1.5.3 — Prometheus Config ✅ 2026-04-12

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

#### Post-Migration InfluxDB Checklist

- Remove all `DOCKER_INFLUXDB_INIT_*` environment variables from stack file after first setup — leaving them risks re-init on a clean volume
- Token stored in InfluxDB BoltDB does not survive a fresh container with new data volume — regenerate and update all clients (Proxmox, Grafana, Telegraf)
- v1 compatibility requires: (1) DBRP mapping via `influx v1 dbrp create`, (2) v1 auth via `influx v1 auth create`
- Grafana requires two datasources for full coverage: InfluxQL (existing dashboards) + Flux (dashboard 15356)
- Traefik metrics in v3: requires `entryPoint: traefik` in `metrics.prometheus` config and `api.insecure: true`

---

### Step 1.5.4 — Promtail Config ✅ 2026-04-12

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

### Step 1.5.5 — Monitoring Stack File ✅ 2026-04-12

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

### Step 1.5.6 — Deploy ✅ 2026-04-17

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

### Step 1.5.7 — Grafana Initial Setup ✅ 2026-04-28

```
Login: admin / admin → change immediately
```

|Source|Type|URL|
|---|---|---|
|Prometheus|Prometheus|`http://prometheus:9090`|
|Loki|Loki|`http://loki:3100`|
|InfluxDB|InfluxDB (Flux)|`http://influxdb:8086` — org `home`, bucket `proxmox_metrics`|

Import dashboards:

| ID    | Name                   | Source     | Status       |
| ----- | ---------------------- | ---------- | ------------ |
| 17346 | Traefik v3 Official    | Prometheus | ✅ 2026-04-18 |
| 1860  | Node Exporter Full     | Prometheus | ✅ 2026-04-18 |
| 15356 | Proxmox Cluster [Flux] | InfluxDB   | ✅ 2026-04-18 |
| 13639 | Loki container logs    | Loki       | ✅ 2026-04-18 |
[!note] Proxmox metrics Proxmox native metric server configured under Datacenter → Metric Server → InfluxDB. Server: `10.0.60.41`, Port: `8086`, Protocol: HTTP, API Version: 2, Org: `home`, Bucket: `proxmox_metrics`. Pushes on 1-minute interval. Known limitation: LXC container disk I/O not reported — upstream Proxmox bug, not fixable in dashboard.


- [x] Admin password changed ✅ 2026-04-12
- [x] Prometheus data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] Loki data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] InfluxDB data source connected and tested [priority:: 2] ✅ 2026-04-17
- [x] Core dashboards imported [priority:: 2] ✅ 2026-04-17
- [x] look at other database migrations [priority:: 2] ✅ 2026-04-18
- [x] Migrate influxDB from old server to new server - old setup to new setup [priority:: 1] ✅ 2026-04-28
- [x] Migrate prometheus from old server to new server - old setup to new setup [priority:: 1] ✅ 2026-04-28

---

### Step 1.5.8 — Hand off to Phase 8

> [!abstract] Core monitoring is now running. Phase 8 ([[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]) covers the full monitoring target build-out. Complete Phase 2 (remaining VMs) first so all scrape targets exist, then work through Phase 8.

Phase 8 adds: Proxmox host metrics (`pve-exporter`), pfSense metrics (SNMP exporter), ~~Alertmanager~~ ✅ deployed 2026-05-20 PR #48, app-specific exporters (Plex, Sonarr, Radarr, Home Assistant, UniFi), Uptime Kuma for external service checks, dashboard refinement and alert rules.

---

## Phase 2 — Remaining VM Provisioning ✅ 2026-04-24

> [!abstract] Deploy remaining worker VMs after monitoring is stable. Having Prometheus and Grafana running means you get instant visibility into each new VM as it joins.

### Step 2.1 — Provision Worker VMs ✅ 2026-04-24

Run from Proxmox host. Provision one at a time and verify each joins cleanly before proceeding.

```bash
# worker-media-01 — Plex + Tautulli only
./02-provision-vm.sh --vlan 50 --cores 4 --memory 8192 \
  --label type=media worker-media-01
./03-post-boot.sh worker-media-01

docker node ls

# Removed 2026-05-24 — never provisioned. Media management role taken by
# worker-mediamanagement-01. Dev/homelab role replaced by dev-node-01.

docker node ls

# worker-controller-01 — dual-homed: VLAN 60 + IoT VLAN 20
./02-provision-vm.sh --vlan 60 --nic2-vlan 20 --cores 2 --memory 4096 \
  --label zone=controller,nas=nfs worker-controller-01
./03-post-boot.sh worker-controller-01
```

> [!important] VLAN 100 NFS — static IP required, no gateway VMs with `--nic2-vlan 100` get a DHCP lease on VLAN 100 but must not accept a default gateway from it. `03-post-boot.sh` writes `/etc/netplan/51-eth1-nas.yaml` with `use-routes: false` and `use-dns: false`. Without this, `dhclient eth1` overwrites the default route and kills SSH. See Gotcha #25.

> [!important] pfSense VLAN 50 → VLAN 60 firewall rules required before provisioning media VMs Swarm join requires TCP 2377, TCP/UDP 7946, UDP 4789 open from VLAN 50 to manager at `10.0.60.30`. Add these rules in pfSense before running `03-post-boot.sh` on any VLAN 50 VM. See Gotcha #28.

> [!note] Portainer stack auto-redeploys after each join `03-post-boot.sh` now handles this automatically — `redeploy_portainer_after_join()` runs after each worker joins so overlay networks propagate correctly.

> [!warning] Auto-join may fail silently (Bug #8) If a VM doesn't appear in `docker node ls`, SSH in and join manually. Apply the node label on `manager-01` afterwards. See Step 1.5.1 for manual recovery commands.

- [x] Go over media and media management setup to cross-check - need to also pay attention to how migration will work ✅ 2026-04-18
- [x] `worker-media-01` provisioned and joined Swarm ✅ 2026-04-18
- [x] `worker-mediamanagement-01` provisioned and joined Swarm ✅ 2026-04-18 — downloads originally on SSD zvol (`rpool/data/vm-204-downloads`), migrated to `MainStorage/downloads` virtiofs share 2026-04-26, zvol destroyed
- [x] `worker-controller-01` provisioned and joined Swarm ✅ 2026-04-24
- [x] All nodes show `Ready` in `docker node ls` ✅ 2026-04-24
- [x] node-exporter and cAdvisor confirmed in Prometheus for `worker-controller-01` [priority:: 3]

### Step 2.2 — dev-node-01 & Obsidian MCP Deployment

#### Prerequisites

- [x] pfSense rule: VLAN 40 ↔ VLAN 60 UDP 4789 (Swarm VXLAN overlay)
- [x] pfSense rule: VLAN 40 ↔ VLAN 60 TCP 2377 (Swarm control plane)
- [x] pfSense rule: VLAN 40 ↔ VLAN 60 TCP/UDP 7946 (Swarm gossip)
- [x] pfSense rule: VLAN 10 → VLAN 40:22000 TCP/UDP (Syncthing sync)
- [x] pfSense rule: VLAN 10 → VLAN 40:22 TCP (SSH dev access)

#### Provisioning

- [x] Provision `dev-node-01` — VLAN 40, 2 vCPU, 4GB RAM, 40GB disk
- [x] Run `03-post-boot.sh` — verify actual IP with `ip addr` before running
- [x] Confirm node appears in Portainer with `zone=dev` label ✅ 2026-05-25

#### MCP server hardening (Claude Code session)

- [x] Add API key middleware to Express `/mcp` route ✅ 2026-05-24 — implemented secure header, query param, and Bearer token parsing
- [x] Add `VAULT_ALLOWED_FOLDERS` env var and enforcement in vault service ✅ 2026-05-24 — structured allowlist matching, filters lists, searches, and tree browsing dynamically
- [x] Add request logging to stdout (Loki picks up via Promtail) ✅ 2026-05-24 — structured JSON stdout logs with timestamp, status, path, and duration
- [x] Push to `scarredNinja/obsidian-mcp-server`, open PR ✅ 2026-05-25

#### Stack deployment

- [x] Write `stack-syncthing.yml` — zone=dev, volume /mnt/docker-data/vault ✅ 2026-05-25
- [x] Configure Syncthing — pair with Windows PC, set `.stignore` ✅ 2026-05-25
- [x] Write `stack-obsidian-mcp.yml` — zone=dev, vault mount :ro, Traefik labels ✅ 2026-05-25
- [x] Deploy both stacks via Portainer ✅ 2026-05-25
- [x] Confirm `obsidian-mcp.home.purvishome.com` resolves and returns 200 ✅ 2026-05-25

#### Validation

- [x] Test Claude Desktop stdio transport locally ✅ 2026-05-25
- [x] Test HTTP endpoint via Traefik from VLAN 10 ✅ 2026-05-25
- [x] Test Antigravity CLI connection to HTTP endpoint ✅ 2026-05-25
- [ ] Confirm credential files absent from synced vault on dev-node-01

---

## Phase 3 — Legacy Service Migration ✅ 2026-05-08

> [!abstract] Strategy Services run as Docker Compose on the old server (same Proxmox host, different ZFS pool). Migration uses **ZFS send/receive** — fast, atomic, and snapshot-consistent. No data crosses the network.

### Migration Overview

| Service        | Old Data Path          | Destination VM              | Downtime |
| -------------- | ---------------------- | --------------------------- | -------- |
| Plex           | `/opt/plex/config`     | `worker-media-01`           | ~5 min   |
| Tautulli       | `/opt/tautulli/config` | `worker-media-01`           | ~2 min   |
| Sonarr         | `/opt/sonarr/config`   | `worker-mediamanagement-01` | ~2 min   |
| Radarr         | `/opt/radarr/config`   | `worker-mediamanagement-01` | ~2 min   |
| Prowlarr       | `/opt/prowlarr/config` | `worker-mediamanagement-01` | ~2 min   |
| Transmission   | `/opt/transmission`    | `worker-mediamanagement-01` | ~2 min   |
| Home Assistant | `/opt/homeassistant`   | `worker-controller-01`      | ~5 min   |
| UniFi          | `/opt/unifi`           | `worker-controller-01`      | ~5 min   |
| Grafana        | `/opt/grafana`         | `worker-monitoring-01`      | ~2 min   |
| Prometheus     | `/opt/prometheus/data` | `worker-monitoring-01`      | ~5 min   |
| InfluxDB       | `/opt/influxdb`        | `worker-monitoring-01`      | ~5 min   |
Migration scripts exist in repo at `scripts/migrate-media.sh` and `scripts/migrate-mediamanagement.sh` (branch `claude/focused-feynman-ea3eba`).


> [!tip] Priority Order Migrate in this order: **Plex → HA → UniFi → Sonarr/Radarr → InfluxDB → Grafana → Prometheus**. Prometheus history is acceptable to lose — it rebuilds within hours.

---

### Step 3.1 — Snapshot All Old Datasets ✅ 2026-04-24

```bash
SNAP="migrate-$(date +%Y%m%d)"

for ds in plex tautulli sonarr radarr transmission prowlarr homeassistant unifi grafana prometheus influxdb; do
  zfs snapshot oldpool/docker/${ds}@${SNAP}
  echo "Snapshot: oldpool/docker/${ds}@${SNAP}"
done
```

- [x] All snapshots created ✅ 2026-04-24
- [x] Verified with `zfs list -t snapshot | grep migrate` ✅ 2026-04-24

---

### Step 3.2 — Stop Services on Old Server ✅ 2026-04-24

```bash
cd /opt/compose
docker compose -f media.yml        down
docker compose -f controllers.yml  down
docker compose -f monitoring.yml   down
```

> [!warning] Home Assistant Shutdown Run `ha core stop` before stopping the HA container. HA uses SQLite — unclean shutdown can corrupt the database.

- [x] Media services stopped ✅ 2026-04-24
- [x] Home Assistant stopped cleanly ✅ 2026-04-24
- [x] UniFi controller stopped ✅ 2026-04-24
- [x] Monitoring stack stopped ✅ 2026-04-24
- [x] All containers confirmed down: `docker ps` returns empty ✅ 2026-04-24

---

### Step 3.3 — ZFS Send/Receive to New Pool ✅ 2026-04-26

[!note] Old data in LXC containers on MainStorage Data is not in ZFS datasets — it lives inside LXC container filesystems on `MainStorage` pool. Import the pool first: `zpool import -f MainStorage`. Data found at:

|Service|Source Path|
|---|---|
|Plex (61G)|`/MainStorage/subvol-101-disk-0/home/docker/plex/`|
|Tautulli|`/MainStorage/subvol-101-disk-0/home/docker/tautulli/`|
|Sonarr|`/MainStorage/subvol-104-disk-0/home/docker/sonarr/`|
|Radarr|`/MainStorage/subvol-104-disk-0/home/docker/radarr/`|
|Prowlarr|`/MainStorage/subvol-102-disk-0/home/ha/docker/`|
|Transmission|`/MainStorage/subvol-102-disk-0/home/ha/docker/transmission/`|

Use `rsync` not `zfs send` — these are plain directories not datasets. **Always run `zpool export MainStorage` before rebooting** — force import overwrites the hostid stamp and the old system will need `-f` to re-import on next boot.

```bash
BASE="/MainStorage"
DEST="/mnt/docker-data"

rsync -av --progress ${BASE}/subvol-101-disk-0/home/docker/plex/     ${DEST}/plex/
rsync -av --progress ${BASE}/subvol-101-disk-0/home/docker/tautulli/ ${DEST}/tautulli/
rsync -av --progress ${BASE}/subvol-104-disk-0/home/docker/sonarr/   ${DEST}/sonarr/
rsync -av --progress ${BASE}/subvol-104-disk-0/home/docker/radarr/   ${DEST}/radarr/

# Prowlarr + Transmission — confirm exact paths first
ls /MainStorage/subvol-102-disk-0/home/ha/docker/ | grep -Ei "prowlarr|transmission"
# Then rsync accordingly
```

- [x] Media datasets transferred ✅ 2026-04-24
- [x] Arr stack datasets transferred ✅ 2026-04-24
- [x] Controller datasets transferred ✅ 2026-04-24
- [x] Monitoring datasets transferred ✅ 2026-04-24
- [x] Downloads (440 GB) transfer to `worker-mediamanagement-01:/mnt/downloads` complete [priority:: 2] — 153G from SSD zvol migrated 2026-04-26 ✅; 440G from `MainStorage/subvol-110-disk-0` mv in progress at session end — verify with `ls /MainStorage/downloads/complete/ | wc -l`, then destroy subvol-110

---

### Step 3.4 — Verify Data on Target VMs ✅ 2026-04-24

```bash
ssh ubuntu@worker-media-01
ls /mnt/docker-data/plex/config/Library
ls /mnt/docker-data/tautulli/config

ssh ubuntu@worker-mediamanagement-01
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

- [x] Plex library visible on `worker-media-01` ✅ 2026-04-19
- [x] Tautulli config visible on `worker-media-01` ✅ 2026-04-24
- [x] Sonarr/Radarr/Prowlarr/Transmission configs visible on `worker-mediamanagement-01` ✅ 2026-04-24
- [x] HA `configuration.yaml` visible on `worker-controller-01` ✅ 2026-04-24
- [x] UniFi data visible on `worker-controller-01` ✅ 2026-04-24
- [x] InfluxDB and Grafana data visible on `worker-monitoring-01` ✅ 2026-04-24

---

### Step 3.5 — Deploy Swarm Stacks ✅ 2026-04-25

```bash
# Create shared media overlay network first — needed for Plex webhook from Arr stack
docker network create --driver overlay --attachable media

docker stack deploy -c /mnt/docker-swarm/stacks/plex/stack-plex.yml         plex
docker stack deploy -c /mnt/docker-swarm/stacks/arr/stack-arr.yml             arr
docker stack deploy -c /mnt/docker-swarm/stacks/controllers/stack-controllers.yml controllers

# 1. Create Docker secrets
printf '%s' 'claim-XXXXXXXX' | docker secret create plex_claim -        # fresh from plex.tv/claim — 4 min TTL
docker secret ls

# NOTE: VPN secrets (vpn_service_provider, vpn_type, wireguard_private_key) are NO LONGER needed in Swarm.
# Transmission + Gluetun run as Docker Compose (compose-vpn.yml) on worker-mediamanagement-01.
# VPN credentials are set directly in compose-vpn.yml environment. See Gotcha #37.

# 2. Confirm media overlay network exists
docker network ls | grep media
# If missing:
docker network create --driver overlay --attachable media

# 3. Confirm node labels
docker node inspect worker-media-01 --pretty | grep -A3 Labels
docker node inspect worker-mediamanagement-01 --pretty | grep -A3 Labels


watch docker stack ps plex
watch docker stack ps arr
watch docker stack ps controllers
```

> [!important] `media` overlay network Must exist before deploying either stack. Plex and Sonarr/Radarr both join it so Sonarr/Radarr can reach `plex:32400` for library refresh webhooks across VM boundaries.

- [x] `media` overlay network created ✅ 2026-04-19
- [x] Plex stack deployed and running on `worker-media-01` ✅ 2026-04-19
- [x] Arr stack deployed and running on `worker-mediamanagement-01` ✅ 2026-04-25 (Gluetun/Transmission split to `compose-vpn.yml` — see Gotcha #37)
- [x] Controller stack fully running — HA + UniFi container health verified on `worker-controller-01` [priority:: 2]

---

### Step 3.6 — Post-Migration Validation

| Service             | Check                                              | Pass?                                                                                     |
| ------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Plex                | Library intact, no missing media                   | ✅ 2026-04-24                                                                              |
| Sonarr              | System → Status: no config errors, series visible  | ✅ 2026-04-30                                                                              |
| Radarr              | System → Status: no config errors, movies visible  | ✅ 2026-04-30                                                                              |
| Sonarr→Plex webhook | Add test item — Plex library updates automatically | ✅ 2026-04-30                                                                              |
| Home Asst.          | States populated, integrations online              | ✅ 2026-04-28 — routing fixed (port mode:host + trusted_proxies), Prometheus scrape active |
| UniFi               | Devices adopted and connected                      | ✅ 2026-04-29                                                                              |
| InfluxDB            | `influx ping` returns OK                           | ✅ 2026-04-24                                                                              |
| Grafana             | Dashboards load, data sources connected            | ⏳ dashboards migrated — data source names need updating                                   |
| Transmission        | Web UI loads, torrent list intact                  | ✅ 2026-04-25 (compose-vpn stack)                                                          |
| Prowlarr            | Indexers connected                                 | ✅ 2026-04-24                                                                              |

---

- [x] Re-adopt UniFi AP to new controller — set inform URL to `http://10.0.60.42:8080/inform` via SSH or device reset [priority:: 2] ✅ 2026-04-29
- [x] Verify UniFi Prometheus metrics — confirm scrape job active and data visible in Grafana [priority:: 2]
- [x] Allow Home VLAN (VLAN 10 — `10.0.10.0/25`) access to Home Assistant — two steps: (1) add pfSense rule VLAN 10 → `10.0.60.42:8123`; (2) add `10.0.10.0/25` to Traefik `internal-only` middleware allowlist in `middlewares.yaml` [priority:: 2] ✅ 2026-05-08 — pfSense rule added; `internal-only` middleware already had `10.0.10.0/24` covering the range. Confirmed in [[Session Notes — 2026-05-08 — Monitoring Fixes, unifi-poller, Git Integration]].

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

### Cloudflare Tunnel — External Access (CGNAT Environment)

> [!important] This homelab is behind ISP CGNAT (pfSense WAN = 100.77.x.x private address).
> Inbound port forwarding is impossible. All external access uses Cloudflare Tunnel.
> See Gotcha #33.

**How it works:** `cloudflared` makes an outbound-only connection to Cloudflare's edge
network. No inbound ports required. Cloudflare terminates TLS for the end user;
cloudflared speaks plain HTTP to the backend service internally.

**Current tunnel routes:**

| Public Hostname | Internal Backend | Notes |
|----------------|-----------------|-------|
| `plex.purvishome.com` | `http://plex:32400` | Direct to Plex, Cloudflare handles TLS |

**Adding a new external service:**
1. Cloudflare Dashboard → Zero Trust → Networks → Tunnels → edit tunnel
2. Add Public Hostname → subdomain, domain, type HTTP, URL = `servicename:port`
3. Cloudflare auto-creates orange-cloud CNAME — no manual DNS needed

**cloudflared deployment:**
- Stack: `stack-plex.yml`, service name `cloudflared`
- Image: `cloudflare/cloudflared:2025.1.0` (pinned — was `:latest`, changed PR #61 2026-05-26)
- Placement: `type=media` (worker-media-01)
- Networks: `media` + `traefik-public`
- Token: stored as Docker secret `cloudflare_tunnel_token` (was Portainer env var `CLOUDFLARE_TUNNEL_TOKEN` — changed PR #61 2026-05-26). Passed to the container via `TUNNEL_TOKEN` env var set from the secret at container start.
- Image is **distroless** — no `/bin/sh`, no debug tools. Entrypoint shims (`sh -c '...'`) are not possible; only `cloudflared` itself can be invoked. See Gotcha #59.

> [!warning] Cloudflare ToS
> Free plan prohibits large media streaming through Cloudflare proxy. Plex is personal
> use / grey area. For high-bandwidth services, prefer direct streaming after auth where
> the application supports it.

---

### Transmission + Gluetun VPN Sidecar

> [!note] Gluetun controls only Transmission's network namespace. All other services on `worker-mediamanagement-01` use normal routing.

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

### Task — Migrate Stack Deployments to Pull Directly from Git Repo

> [!important] Currently stacks are deployed by manually copying YAML files to `/mnt/docker-swarm/stacks/` on the Proxmox host virtiofs share. This creates drift risk — the running stack and the repo can diverge silently.

**Target state:** All stacks deploy directly from the GitHub repo, removing the manual copy step.

**Options (evaluate in order of simplicity):**

**Option A — Portainer Git integration (recommended)**
Portainer CE supports deploying stacks directly from a Git repo. Per-stack configuration:
- Repository URL: `https://github.com/scarredNinja/docker-swarm-home`
- Branch: `main`
- Compose file path: `proxmox-swarm/stacks/<stack>/<file>.yml`
- Enable "Auto update" with webhook or polling interval

This eliminates manual copy entirely. Stack redeployments happen via Portainer UI
or Git webhook — the repo is always the source of truth.

**Option B — deploy script on manager-01**
Keep manual copy but automate it: a script that `git pull` on manager-01 and
copies updated stack files to the virtiofs mount, then redeploys changed stacks.
Simpler than Portainer Git but still requires manual trigger.

- [x] Add Portainer standalone environment for `worker-mediamanagement-01` ✅ 2026-05-07 — docker-socket-proxy deployed on port 2375, registered in Portainer as Docker API environment. Provides container visibility and management for `compose-vpn.yml` containers. pfSense rule: `10.0.60.30 → 10.0.50.51:2375` TCP. Stack deployment via Portainer not possible on Swarm worker nodes (see Gotcha #54). `compose-vpn.yml` had `restart: unless-stopped` but did not survive VM reboot — Swarm overlay gossip race meant attach failed before Docker gave up restart. Replaced with systemd unit `compose-vpn.service` ✅ 2026-05-08 (PR #25). See Gotcha #58.
- [x] Evaluate Portainer Git stack integration for each stack [priority:: 3] #backlog ✅ 2026-05-08
- [x] Configure Portainer Git integration for `traefik` stack [priority:: 3] #backlog
- [x] Configure Portainer Git integration for `controller` stack [priority:: 3] #backlog ✅ 2026-05-08
- [x] Configure Portainer Git integration for remaining stacks [priority:: 3] #backlog ✅ 2026-05-08
- [x] Remove `deploy-traefik-config.sh` manual copy step once Git integration confirmed [priority:: 3] #backlog ✅ 2026-05-08

---

## Phase 9 — Admin VPN (Tailscale)

> [!abstract] Status
> ✅ **Complete** — Tailscale deployed and routing to VLAN 60 confirmed 2026-05-08.
> Traefik `internal-only` middleware updated, all admin devices can reach VLAN 60 services via Tailscale.
> See [[Session Notes — 2026-05-07 — Tailscale Admin VPN]] and `docs/admin-vpn.md` in repo.

### Problem Statement

This homelab is behind ISP CGNAT (pfSense WAN = `100.77.x.x`). Inbound port forwarding is impossible. Cloudflare Tunnel handles public access for Plex but is not appropriate for admin services. Goal: secure off-site access to VLAN 60 services for admin use without exposing anything publicly.

### Solution — Tailscale + pfSense Subnet Router

**Architecture:**

```
Admin device (anywhere)
    ↓ Tailscale client
Tailscale DERP relay  ← handles CGNAT traversal, no inbound ports needed
    ↓ encrypted tunnel
pfSense (Tailscale package, subnet router)
    ↓ subnet route: 10.0.60.0/25
VLAN 60 services (Traefik, Portainer, Grafana, etc.)
```

**Why pfSense as subnet router:**
- No additional VM or container needed
- pfSense already controls VLAN 60 routing
- Subnet router model: Tailscale devices see `10.0.60.0/25` as directly routable

### What's In the Repo

- `config/traefik/dynamic/middlewares.yaml` — `100.64.0.0/10` added to `internal-only` IPAllowList sourceRange ✅ deployed
- `docs/admin-vpn.md` — full architecture, implementation checklist, security notes, upgrade path

### What's Manual (pfSense + Tailscale admin)

| Task | Location | Status |
|---|---|---|
| Install Tailscale package | pfSense → System → Package Manager | ✅ Done |
| Register pfSense to Tailscale account | Tailscale admin console | ✅ Done |
| Advertise subnet `10.0.60.0/25` | pfSense Tailscale settings | ✅ Done |
| Approve subnet route | Tailscale admin console → Machines | ✅ Done |
| pfSense firewall rule: Tailscale → VLAN 60 | pfSense → Firewall → Rules → Tailscale | ✅ Done |
| Enrol client devices | Tailscale client apps | ✅ Phone done |
| ACL policy — restrict to VLAN 60 only | Tailscale admin → Access Controls | ⏳ Pending |

### Implementation Checklist

- [x] Phase A — pfSense setup: install package, register, advertise `10.0.60.0/25`, approve route
- [x] Phase B — pfSense firewall: Tailscale → VLAN 60 rule
- [x] Phase C — Client enrolment: phone enrolled, subnet route enabled
- [x] Phase D — Traefik: `100.64.0.0/10` added to `internal-only` allowlist, deployed
- [x] Phase E — End-to-end test confirmed ✅ 2026-05-08 — Tailscale deployed and routing to VLAN 60

### Security Considerations

- Tailscale ACL should restrict to `10.0.60.0/25` only — no Tailscale device should reach IoT/home VLANs
- Enable key expiry on all client devices
- MFA/SSO: Tailscale supports Google/GitHub SSO — enable for the account
- Split tunnel is default — only `10.0.60.0/25` routes via Tailscale, not all traffic

### Upgrade Path

| Option | When | Notes |
|---|---|---|
| **Headscale** | If Tailscale pricing/ToS changes | Self-hosted coordination server; same WireGuard plane |
| **Authelia + passkeys** | Phase 10 | Application-level auth over Tailscale; replaces per-app credentials |
| **Tailscale ACLs** | Now | Restrict device-to-subnet access rules in admin console |

## Scripts Reference

This section provides a quick reference for the custom automation and utility scripts located under the `scripts/` directory of the `docker-swarm-home` repository.

| Script | Purpose |
|---|---|
| `scripts/create-alertmanager-secret.sh` | Interactively prompts for and creates the `discord_webhook_url` Docker secret for Alertmanager without saving it to shell history. |
| `scripts/create-monitoring-network.sh` | Idempotently creates the shared `monitoring` internal overlay network (subnet `10.200.1.0/24`) for secure, isolated metrics scraping between Swarm stacks. |
| `scripts/deploy-alertmanager.sh` | Deploys or updates the monitoring stack with Alertmanager on `manager-01`, verifying config prerequisites and managing the Discord webhook secret. |
| `scripts/deploy-backup-scripts.sh` | Installs backup utility scripts and schedules overnight backup cron jobs (Proxmox config, ZFS snapshots, vzdump VM backups, Restic backups, and disk space checks) on the Proxmox host. |
| `scripts/deploy-worker-scripts.sh` | Deploys `nfs-bench.sh` and its associated cron job to target Swarm worker VMs (locally or remotely via SSH from `manager-01`) and seeds initial performance metrics. |
| `scripts/disk-space-check.sh` | Monitors free space on ZFS datasets (MainStorage pool) and remote Synology storage, writing Prometheus metrics and firing Discord alerts when thresholds are crossed. |
| `scripts/nfs-bench.sh` | Runs comprehensive network and storage performance benchmarks (iperf3, dd, fio) against remote Synology NFS mounts and exports results to stdout, JSON log, and Prometheus. |

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

> [!note] Updated 2026-05-07 — PBS decided against; VM backups via vzdump instead.

| Layer | Tool | Covers | Schedule |
|---|---|---|---|
| Host config backup | `proxmox-config-backup.sh` | `/etc/pve`, network, cron, fstab, `/usr/local/bin` | Daily 01:00 |
| Data recovery | `zfs-snapshot.sh` + ZFS send | All four docker datasets → `MainStorage/backups/` | Daily 02:15 |
| VM recovery | `vm-backup.sh` (vzdump) | All 6 VMs, stop mode, 3 copies | Daily 03:00 |
| Off-site | `restic-backup.sh` → restic-rest-server | All of `MainStorage/backups/` → Synology NFS | Daily 05:00 |
| Point-in-time | ZFS snapshot (manual) | Before any major change | On demand |

**restic-rest-server:** Deployed as Swarm stack (`stacks/stack-restic.yml`), pinned to `zone=monitoring`, NFS mount from Synology (`10.0.100.20:/volume1/docker-backups`, nfsvers=3), port 8000 via routing mesh. Proxmox host requires pfSense rule: `10.0.90.50 → 10.0.60.0/24:8000`. As of PR #61 (2026-05-26): `--no-auth` replaced with `--htpasswd-file /run/secrets/restic_htpasswd`; credentials stored as Docker secret `restic_htpasswd` (bcrypt htpasswd file). The `RESTIC_REPO` on the Proxmox host now includes `restic:<password>@` credentials in the URL.

- [x] Build Grafana backup dashboard ✅ 2026-05-07 — PR #18. `write_prom_metrics` added to all 4 scripts, node_exporter textfile collector configured via `deploy-backup-scripts.sh`, Prometheus scrape job at `10.0.90.50:9100`, dashboard at `proxmox-swarm/stacks/monitoring/dashboards/backup-status.json`. Import into Grafana manually. `proxmox_config_backup` confirmed; remaining 3 jobs confirm after overnight cron run.
- [x] Add pfSense rule: VLAN 60 → `10.0.90.50:9100` TCP — Prometheus scrape of Proxmox host node_exporter [priority:: 2] ✅ 2026-05-08
- [x] Verify all 4 backup jobs visible in Grafana backup dashboard after overnight run [priority:: 2] ✅ 2026-05-08

---

## Appendix C — Stack File Reference

All stack files live in the [GitHub repo](https://github.com/scarredNinja/docker-swarm-home) under `proxmox-swarm/stacks/`.

| Stack       | File path                                  | Pinned to              | Status                 |
| ----------- | ------------------------------------------ | ---------------------- | ---------------------- |
| Portainer   | `stacks/portainer/stack-portainer.yml`     | `node.role==manager`   | ✅ Deployed             |
| Monitoring  | `stacks/monitoring/stack-monitoring.yml`   | `zone=monitoring`      | ✅ Deployed             |
| Plex        | `stacks/plex/stack-plex.yml`               | `type=media`           | 🔧 Built, not deployed |
| Arr         | `stacks/arr/stack-arr.yml`                 | `zone=mediamanagement` | 🔧 Ready, not deployed |
| Controllers | `stacks/controller/stack-controller.yml`   | `zone=controller`      | ✅ Deployed — unpoller added |

---

Copy from repo to virtiofs share before deploying:

```bash
# On Proxmox host — run after any repo update
bash /root/copy-traefik-config.sh
```

---

## Appendix D — Gotchas Reference

> [!abstract] Hard-won fixes. Each entry has the symptom, root cause, and fix. Entries marked `03-post-boot.sh` are now automated — manual steps only needed outside normal provisioning flow.

| #   | Area                    | Symptom                                                                     | Root Cause                                                                                                                                                                 | Fix                                                                                                                                                                                                                           |
| --- | ----------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | cloud-init              | `cicustom` vendor config silently ignored                                   | Ubuntu 24.04 defaults to NoCloud datasource                                                                                                                                | Inject `99-pve.cfg` into image via `qemu-nbd` before Proxmox import                                                                                                                                                           |
| 2   | cloud-init              | Setting `cicustom user=` separately drops `vendor=`                         | Proxmox bug                                                                                                                                                                | Always set `cicustom` as `user=...,vendor=...` in a single `qm set` call                                                                                                                                                      |
| 3   | Docker storage          | Overlay2 driver fails on virtiofs mounts                                    | virtiofs lacks `d_type` support                                                                                                                                            | Use `fuse-overlayfs` storage driver in `/etc/docker/daemon.json`                                                                                                                                                              |
| 4   | Bash scripting          | Script crashes when loop index hits 0                                       | `(( idx++ ))` returns exit code 1, triggers `set -e`                                                                                                                       | Use `idx=$(( idx + 1 ))` instead                                                                                                                                                                                              |
| 5   | cloud-init              | UTF-8 chars or CRLF in heredocs break YAML silently                         | Windows line endings or special chars                                                                                                                                      | Strip with `sed 's/\r//'`; avoid non-ASCII in heredocs                                                                                                                                                                        |
| 6   | ZFS                     | Mounting ZFS zvols via kpartx corrupts EXT4                                 | ZFS and kpartx incompatible                                                                                                                                                | Use `qemu-nbd` on source image instead                                                                                                                                                                                        |
| 7   | Docker networking       | `dockerd` fails: `cannot create network ... networks have same bridge name` | Ghost bridge interfaces and stale `local-kv.db` survive Docker crash                                                                                                       | Pre-flight bridge flush + `rm -rf /var/lib/docker/network/files/local-kv.db`. Baked into `03-post-boot.sh`.                                                                                                                   |
| 8   | Portainer               | Agent only schedules on manager; workers reject scoped network              | Overlay network not propagated to nodes that joined after initial stack deploy                                                                                             | Redeploy Portainer stack after all nodes joined. `03-post-boot.sh` handles this automatically.                                                                                                                                |
| 9   | SSH                     | `BatchMode=yes` fails with passphrase-protected key                         | Passphrase requires interactive input                                                                                                                                      | Use passphrase-free SSH keys for automation                                                                                                                                                                                   |
| 10  | virtiofs                | Containers can't access virtiofs-mounted directories                        | Incorrect ownership on host-side ZFS dataset                                                                                                                               | `chown -R <uid>:<gid> /path/to/dataset` on Proxmox host                                                                                                                                                                       |
| 11  | Traefik                 | Container fails to start: `acme.json: permission denied`                    | `acme.json` must exist with `chmod 600` before Traefik starts                                                                                                              | `touch acme.json && chmod 600 acme.json`                                                                                                                                                                                      |
| 12  | Provisioning            | `CI_USER` env var mismatches across scripts                                 | Default not persisted between shell sessions                                                                                                                               | Always `export CI_USER=ubuntu` explicitly                                                                                                                                                                                     |
| 13  | Proxmox API             | `qm agent` JSON parsing fails                                               | API returns plain array, not `{"result":[]}`                                                                                                                               | Parse raw array directly                                                                                                                                                                                                      |
| 14  | DHCP                    | VM gets no IP after provisioning                                            | VLAN tag not set on NIC                                                                                                                                                    | Set VLAN tag explicitly in `02-provision-vm.sh`                                                                                                                                                                               |
| 15  | pfSense DynDNS          | `Could not route to /client/v4/zones/token/dns_records`                     | pfSense CE 2.8.0 substitutes Username field literally into Cloudflare API URL                                                                                              | Use Zone ID as Username field. Get from Cloudflare Dashboard → domain → Overview → right sidebar.                                                                                                                             |
| 16  | Traefik v3 + Docker 29  | Traefik rejects API 1.24                                                    | Docker 29.4 dropped support for old API versions                                                                                                                           | Originally used `traefik:latest` to avoid older pinned tags. As of PR #61 (2026-05-26) all images are pinned to specific stable versions — use the pinned tag from the stack file.                                            |
| 17  | Traefik on Swarm worker | Traefik can't list Swarm services                                           | Worker nodes can't query Swarm API from local Docker socket                                                                                                                | Deploy `tecnativa/docker-socket-proxy` on manager, connect via `traefik-backend` overlay. Set endpoint to `tcp://docker-proxy:2375`.                                                                                          |
| 18  | Traefik entrypoint name | `EntryPoint doesn't exist: https` in dynamic config                         | Entrypoint renamed from `https` to `websecure` in updated config                                                                                                           | Update all dynamic config files: `entrypoints: [websecure]` not `[https]`                                                                                                                                                     |
| 19  | virtiofs + Docker       | Swarm manager demotes itself to worker on reboot                            | Docker creates data subdirs with `700` on host-side ZFS mountpoint. virtiofs exposes host permissions directly — no remapping. Docker can't read Raft state DB on restart. | `chmod -R 755 /mnt/docker-data/` on Proxmox host after first Docker start. Must be host-level — VM-side fixes don't survive Docker restart. Automated in `03-post-boot.sh`.                                                   |
| 20  | virtiofs ownership      | Containers fail silently on virtiofs-mounted paths                          | virtiofs exposes host-side ownership directly. Directories created by a previous user are inaccessible.                                                                    | `chown -R root:root` on affected paths on Proxmox host. Verify with `ls -la /mnt/docker-swarm/`.                                                                                                                              |
| 21  | Swarm state DB          | `Swarm: pending` / `Is Manager: false` after reboot or server switch        | `docker-state.json` persists on virtiofs share with empty `AdvertiseAddr` — stale join-in-progress state from interrupted init survives VM rebuild                         | Clear `/mnt/docker-data/swarm/*` and `/mnt/docker-data/network/files/local-kv.db` on Proxmox host before re-init. `chmod -R 755 /mnt/docker-data/` after. Automated in `03-post-boot.sh`. See Appendix E for manual recovery. |
| 22  | Traefik internal routes | Internal services reachable from DMZ                                        | DMZ VLAN 80 included in `internal-only` middleware IP allowlist                                                                                                            | Remove VLAN 80 from internal allowlist. Only LAN + VLANs 20/40/50/60 should be in the allowlist.                                                                                                                              |
| 23  | Provisioning auto-join  | Worker VM provisioned but does not join Swarm                               | `03-post-boot.sh` does not verify join success and does not apply node labels on manager post-join                                                                         | Manual recovery: join on worker, apply label on manager. Claude Code task raised to fix script. See Step 1.5.1.                                                                                                               |
| 24  | Networking              | Dual-homed VM eth1 gets no IP — DHCP Discover never reaches pfSense         | VLAN tag not trunked correctly from Proxmox bridge to the VM, or pfSense VLAN interface not active                                                                         | Verify: (1) `qm config <vmid> \| grep net1` shows `tag=100`, (2) Extreme switch trunk port includes VLAN 100 tagged, (3) pfSense VLAN interface is enabled. Re-run `03-post-boot.sh --skip-docker --skip-swarm` after fix     |
| 25  | Networking              | `dhclient eth1` on storage VLAN kills SSH — default route overwritten       | pfSense DHCP on VLAN 100 pushes a gateway; `dhclient` replaces existing default route                                                                                      | Set gateway field blank in pfSense DHCP scope for VLAN 100. Write `/etc/netplan/51-eth1-nas.yaml` with `dhcp4-overrides: use-routes: false, use-dns: false`. Automated in `03-post-boot.sh`.                                  |
| 26  | Netplan                 | `use-gateway: false` causes netplan apply error                             | `use-gateway` is not a valid netplan override key                                                                                                                          | Use `use-routes: false` only — this covers gateway suppression. `use-gateway` does not exist in netplan.                                                                                                                      |
| 27  | NFS                     | NFS mount fails — `permission denied` or path not found                     | Synology NFS export path is case-sensitive; default assumption was lowercase                                                                                               | Correct path is `TVShows`, `Movies`, `Animation` (capitalised). Always verify with `showmount -e <nas-ip>` before mounting.                                                                                                   |
| 28  | pfSense firewall        | `worker-media-01` fails to join Swarm — TCP 2377 refused                    | VLAN 50 → VLAN 60 firewall rules missing. Swarm control plane requires TCP 2377, TCP/UDP 7946, UDP 4789 from VLAN 50 to manager at `10.0.60.30`                            | Add pfSense rules before provisioning any VLAN 50 VM. Rules must be in place before `03-post-boot.sh` runs.                                                                                                                   |
| 29  | Traefik                 | acme.json must be chmod 600 on Proxmox host before Traefik starts           | Traefik exits immediately if permissions are wrong.                                                                                                                        | virtiofs exposes host-side permissions directly so the fix must be on the Proxmox host, not inside the VM.                                                                                                                    |
| 30  | Sonarr                  | Sonarr v4 dropped Basic auth                                                | Migrated configs with `AuthenticationMethod: Basic` prevent the web server from binding.                                                                                   | Change `<AuthenticationMethod>Basic</AuthenticationMethod>` to `Forms` in `config.xml` inside the container, then restart the service.                                                                                        |
| 31  | Networking — multi-homed VM | HTTPS connections fail ~50% randomly; TLS handshake never completes from client | Dual default routes at equal metric 100 on `traefik-dmz-01` — `eth0` VLAN 60 + `enp6s19` VLAN 80. Linux ECMP splits SYN-ACK replies: ~50% leave via DMZ interface bypassing pfSense, client never receives completion | Temp: `sudo ip route del default via 10.0.80.1 dev enp6s19`. Permanent: `dhcp4-overrides: use-routes: false` on `enp6s19` in netplan — same pattern as VLAN 100 NICs (Gotcha #25). Temp fix does not survive reboot.          |
| 32  | Docker overlay networking | Service-to-service connections fail; containers cannot reach physical `10.0.x.x` hosts | Docker auto-assigns overlay subnets from `10.0.0.0/8`, colliding with physical VLANs. `monitoring_monitoring` was `10.0.4.0/24` (workstation LAN collision); `traefik-public` was `10.0.2.0/24` | Pin subnets in stack YAML `networks.<name>.ipam.config.subnet`. Use `10.200.x.0/24` for all overlays. Current: `monitoring_monitoring`→`10.200.1.0/24`, `traefik-public`→`10.200.2.0/24`, `traefik_traefik-backend`→`10.200.3.0/24` |
| 33  | CGNAT / External Access | Inbound port forwarding silently fails — connections time out, pfSense WAN rules show 0 states | ISP uses Carrier-Grade NAT (100.64.0.0/10); pfSense WAN is a private address; packets never reach pfSense from internet | Use Cloudflare Tunnel (outbound-only connection). Remove pfSense port forward rules — they have no effect behind CGNAT. |
| 34  | Traefik / arr stack | Sonarr 404 — router enabled, service UP, but connection refused on backend | `loadbalancer.server.port` set to `9898` (SslPort from config.xml) not `8989` (Port). Traefik health check passes VIP but routes to wrong port. | Correct label to `8989`. Always verify port via `netstat -tlnp` inside container, not config file field names. |
| 35  | Swarm overlay / pfSense | VXLAN overlay mesh fails between nodes on different VLANs — peers list missing nodes, overlay traffic fails | pfSense blocking UDP 4789 inter-VLAN. VXLAN requires bidirectional UDP 4789 between all Swarm node IPs. Also requires TCP/UDP 7946 (gossip) and TCP 2377 (control plane). | Add pfSense rules on each VLAN hosting Swarm nodes: allow UDP 4789, TCP/UDP 7946, TCP 2377 to all other Swarm VLANs. After fix: `docker service update --force <service>` to trigger re-registration. |
| 36  | virtiofs / Migration    | Container fails to start with permission denied after rsync migration to virtiofs-backed dataset | rsync preserves source ownership (root:root); virtiofs exposes host-side ZFS permissions directly — no UID remapping | `chown -R <uid>:<gid>` on Proxmox host for each service before first container start. See detailed entry below for per-service UIDs. |
| 37  | Docker Swarm / VPN      | `network_mode: service:gluetun` — Transmission starts but traffic not tunnelled; other services can't resolve `gluetun:9091` | `network_mode: service:X` is a Docker Compose feature that shares network namespaces; silently ignored in Swarm — each service keeps its own namespace | Remove Gluetun sidecar. Use pfSense policy routing at network layer — route worker VM's IP through VPN interface. Transparent to all containers. |
| 38  | UniFi                   | `unifi-network-application` logs `*** No MONGO_HOST set ***` and fails to configure | `lscr.io/linuxserver/unifi-network-application` requires external MongoDB and `MONGO_HOST` env var; `unifi-controller` bundles MongoDB internally | Use `unifi-controller` image for bundled MongoDB. To migrate to `unifi-network-application`: export UniFi backup, deploy with MongoDB sidecar, restore via UI. |
| 39  | LXC / Migration         | rsync pulls from wrong path — expected files missing on target VM | Old LXC containers have no standard data root: CT101/103/104/110 use `/home/docker/`, CT102 uses `/home/ha/docker/`, CT105 uses `/home/` flat | Scan all subvolumes with loop before assuming paths; use `ls $subvol/home/` per CT. See detailed entry below. |
| 45  | Docker Swarm / Networking | `network_mode: host` set on Swarm service — container starts but ports not reachable on host NIC | `network_mode: host` is silently ignored in Swarm; service always gets its own network namespace | Publish required ports explicitly with `mode: host` in the `ports` stanza. See detailed entry below. |
| 46  | Home Assistant / Traefik | HA returns HTTP 400 for all Traefik-proxied requests | HA enforces a trusted proxy allowlist; requests without a matching `trusted_proxies` entry are rejected | Add `homeassistant.trusted_proxies` block to `configuration.yaml` covering Docker overlay + Traefik ingress subnets. See detailed entry below. |
| 47  | Prometheus / HA          | HA Prometheus bearer token committed to git or visible in `docker inspect` | `bearer_token:` inline in `prometheus.yml` exposes the token in config files and container metadata | Store token as Docker secret; reference via `bearer_token_file: /run/secrets/ha_prometheus_token`. See detailed entry below. |
| 48  | Backup / cron | `vm-backup.sh` had no cron entry — never ran in production | Script deployed but `deploy-backup-scripts.sh` not re-run after cron section was added | Re-run `deploy-backup-scripts.sh` to atomically write all cron entries |
| 49  | NFS / Go runtime | NFS `hard` mount blocks all Go goroutines under backup load | Go runtime parks goroutines indefinitely on `hard` NFS stall — no timeout = no recovery path | Use `soft,timeo=50,retrans=3`: 5s per attempt × 3 retries before returning error to caller. Note: this partially reverts the Gotcha #68 fix (`hard` mount) — documents the correct tradeoff between EIO risk and goroutine starvation. |
| 50  | Traefik / ACME | `acme.json` has permissions `755` — Traefik logs `"ACME resolve is skipped... permissions 755 are too open"` and issues NO new certs silently | virtiofs exposes host-side file permissions directly into containers. If `acme.json` was created or copied with `755`, Traefik refuses to load the ACME resolver on startup. Existing certs continue to work but no new certs are issued. | `sudo chmod 600 /mnt/docker-data/traefik/data/acme.json` on traefik-dmz-01 (or Proxmox host), then `docker service update --force traefik_traefik`. This is the VM-side fix — virtiofs reflects chmod back to the host dataset. |

---

### Gotcha #36 — virtiofs permissions after rsync (post-migration chown required)

After rsyncing data from an old server to a virtiofs-backed ZFS dataset, the data
lands as `root:root`. virtiofs exposes host-side ZFS permissions directly into
containers with no UID remapping. Containers running as non-root UIDs will fail
to start with permission denied errors.

**Fix:** After any rsync migration, chown on the Proxmox host before starting the container:

```bash
# Prometheus (runs as nobody — UID 65534)
chown -R 65534:65534 /mnt/docker-tsdb/prometheus/

# Grafana (runs as UID 472)
chown -R 472:472 /mnt/docker-data/grafana/

# UniFi controller (runs as UID 1000)
chown -R 1000:1000 /mnt/docker-db/unifi/

# General rule: check container UID with:
# docker inspect <image> | grep -i user
# or check linuxserver.io docs for PUID default
```

Add to post-migration checklist: always chown before first container start.

---

### Gotcha #37 — `network_mode: service:X` not supported in Docker Swarm

`network_mode: service:gluetun` is a Docker Compose feature that shares network
namespaces between two containers on the same host. In Docker Swarm mode this
setting is silently ignored — each service always gets its own network namespace
regardless of this config.

**Symptoms:**
- Dependent service (e.g. Transmission) appears to start but traffic is NOT
  tunnelled through the VPN sidecar
- Other services cannot resolve the sidecar's DNS name (e.g. `gluetun:9091`)
  because the sidecar has no `networks:` stanza

**Fix for VPN in Swarm:** Use pfSense policy routing at the network layer instead
of a container-level VPN sidecar. Route all traffic from the worker VM's IP
through the VPN interface in pfSense. This is transparent to all containers and
works correctly with Swarm networking.

**Fix for DNS resolution:** Ensure all services that need to communicate are on
a shared named overlay network. Every service that needs to be reachable by
others must have an explicit `networks:` stanza.

---

### Gotcha #38 — `unifi-network-application` requires external MongoDB

`lscr.io/linuxserver/unifi-network-application` (the current LinuxServer image)
requires an external MongoDB container and `MONGO_HOST` env var. It will log
`*** No MONGO_HOST set ***` and fail to configure.

The older `lscr.io/linuxserver/unifi-controller` image bundles MongoDB internally
and requires no external database. If migrating data from a `unifi-controller`
setup, use the bundled image — the data formats are not directly compatible.

**Migration path to `unifi-network-application`:**
1. Export UniFi backup: Settings → Maintenance → Backup
2. Deploy `unifi-network-application` with MongoDB sidecar
3. Restore backup via UniFi UI
See commented-out section in `stack-controller.yml` for full MongoDB sidecar config.

---

### Gotcha #39 — LXC container data paths are not standardised

Old LXC containers running Docker Compose do not follow a single convention
for where compose project data lives. During MainStorage migration, paths found:

| CT | Data root | Notes |
|----|-----------|-------|
| 101, 103, 104, 110 | `/home/docker/` | Most common |
| 102 | `/home/ha/docker/` | HA-specific user |
| 105 | `/home/` (flat) | Second disk, each service in own dir |

**Scan command before assuming paths:**
```bash
for subvol in /MainStorage/subvol-*-disk-0; do
  ctid=$(echo $subvol | grep -oP '(?<=subvol-)\d+(?=-disk)')
  echo "=== CT $ctid ==="
  ls $subvol/home/ 2>/dev/null
  echo ""
done
```

Also: CT 110 had Prowlarr config at the `/home/docker/` root level mixed with
other services. Use rsync include/exclude to copy only Prowlarr files selectively.

---

### Gotcha #40 — pmxcfs I/O errors under full disk

**Area:** Proxmox / pmxcfs

**Symptom:** `pvesm add`, `qm set`, file writes to `/etc/pve` fail with "Input/output error" and lock timeout

**Root Cause:** pmxcfs FUSE filesystem becomes unstable when underlying rpool is full — cannot write config changes

**Fix:** Recover disk space first, then `systemctl restart pve-cluster`. All config operations resume normally.

---

### Gotcha #41 — Stale fstab entries after disk migration

**Area:** VM / fstab

**Symptom:** VM fails to mount or has conflicting mount entries; old UUID ext4 entry persists after zvol destroyed

**Root Cause:** fstab entries added incrementally across a session without cleaning up the old disk entry. Old SSD entry (UUID=...) remained after the zvol was destroyed from rpool.

**Fix:** After any disk migration: explicitly remove the old fstab entry, run `systemctl daemon-reload`, verify with `mount | grep <mountpoint>`. Keep only the new virtiofs entry.

---

### Gotcha #42 — Virtiofs directory mapping written twice via pmxcfs

**Area:** Proxmox / virtiofs

**Symptom:** VM cloud-init error on boot: "More than one directory mapping for node pve"

**Root Cause:** Write to `/etc/pve/mapping/directory.cfg` appeared to fail (pmxcfs I/O error) but actually succeeded. Retry after pmxcfs restart wrote a second identical entry.

**Fix:** Edit `/etc/pve/mapping/directory.cfg` manually, remove the duplicate entry. Each mapping block must appear exactly once. Always `cat` the file after writing to confirm single entry before attaching to VM.

---

### Gotcha #43 — Service missing from traefik-public network blocks Traefik routing

**Area:** Traefik / Docker Swarm

**Symptom:** Service URL unreachable via Traefik despite correct labels — router shows UP in dashboard but backend returns connection refused.

**Root Cause:** Service only attached to an `internal: true` overlay network. Traefik cannot reach backends on internal-only networks even if labels are correct.

**Fix:** Add `traefik-public` to the service's networks list in the stack file. Any service requiring Traefik routing MUST be on `traefik-public`.

---

### Gotcha #44 — Traefik v3 metrics endpoint returns 404

**Area:** Traefik / Prometheus

**Symptom:** Prometheus scraping `tasks.traefik_traefik:8080/metrics` returns 404 despite Traefik running.

**Root Cause:** Three issues: (1) `api.insecure: false` disables port 8080 entirely; (2) `metrics.prometheus` missing `entryPoint: traefik` — v3 requires explicit entrypoint; (3) rogue config file in dynamic directory overriding static config.

**Fix:** Set `api.insecure: true`; add `entryPoint: traefik` to metrics.prometheus config; audit `/etc/traefik/dynamic/` for stale files after any config reorganisation.

---

### Gotcha #45 — `network_mode: host` silently ignored in Docker Swarm

`network_mode: host` in a Docker Swarm service definition is silently ignored at
runtime — each service always gets its own network namespace regardless of this config.
The container starts normally with no error, but is on the overlay network, not host.

**Symptoms:**
- Container appears healthy but ports are not reachable on the host NIC
- Services expecting to bind directly to the host interface cannot do so

**Fix:** Explicitly publish required ports in `mode: host`:
```yaml
ports:
  - target: 8123
    published: 8123
    protocol: tcp
    mode: host
```

`mode: host` on a published port bypasses the Swarm mesh/DNAT and publishes
directly on the host NIC. This is the correct approach for services that need
source IP preservation or direct host network access.

**Note:** `network_mode: host` IS honoured when running via `docker compose`
directly on a VM — only ignored when deployed as a Swarm service. If you need
true host networking, run the service via compose rather than Swarm.

---

### Gotcha #46 — Home Assistant returns HTTP 400 for Traefik-proxied requests

HA enforces a trusted proxy allowlist. Requests arriving via Traefik without
a matching `trusted_proxies` entry are rejected with HTTP 400 and logged as:
`"Request does not originate from an allowed source"`

**Fix:** Add to `/mnt/docker-data/homeassistant/configuration.yaml`:
```yaml
homeassistant:
  trusted_proxies:
    - 172.0.0.0/8      # Docker overlay network range
    - 10.0.60.0/24     # Management VLAN (Traefik ingress IP)
  ip_ban_enabled: false
  login_attempts_threshold: -1
```

Restart the HA container after editing. The `trusted_proxies` list must cover
the IP range that Traefik uses when forwarding requests — check the overlay
subnet assigned to `traefik-public` if the above ranges don't match.

---

### Gotcha #47 — Prometheus bearer token for HA must be a Docker secret

The Home Assistant Prometheus integration requires a long-lived access token.
This token must **never** be committed to git or stored in a stack YAML file.

**Correct approach — Docker secret:**
```bash
# Generate token in HA: Profile → Security → Long-Lived Access Tokens
printf '%s' '<your-token>' | docker secret create ha_prometheus_token -
```

Reference in `stack-monitoring.yml` Prometheus service:
```yaml
secrets:
  - ha_prometheus_token
```

And in `prometheus.yml` scrape config:
```yaml
scrape_configs:
  - job_name: 'home_assistant'
    scrape_interval: 60s
    metrics_path: /api/prometheus
    bearer_token_file: /run/secrets/ha_prometheus_token
    static_configs:
      - targets: ['10.0.60.42:8123']
```

**Never use `bearer_token:` inline** — that value appears in `docker inspect`
output and in any config file committed to git.

---

### Gotcha #48 — UniFi adoption fails via Ubiquiti cloud relay behind CGNAT

**Area:** UniFi / Networking

**Symptom:** Adoption stuck indefinitely. Controller logs show repeated WebRTC,
STUN, and TURN failures to Ubiquiti cloud IPs (`162.159.207.0`, `141.101.90.1`).
Error pattern: `TURN instance failed`, `STUN timed out`, `webRtcId terminated`.

**Root Cause:** Ubiquiti's controller uses WebRTC/STUN/TURN cloud relay for AP
adoption. CGNAT blocks the required UDP punch-through — packets never reach
the controller from Ubiquiti's relay.

**Fix:**
1. Factory reset the AP (hold reset button 10 seconds)
2. Disable Remote Access: Settings → System → Advanced → uncheck Enable Remote Access
3. SSH to AP after reset (credentials: `ubnt`/`ubnt`)
4. Run `set-inform http://<controller-ip>:8080/inform` directly
5. AP appears as Pending Adoption — click Adopt in UI

**Prevention:** Always disable Remote Access on self-hosted UniFi controllers.
Local-only adoption via direct inform URL always works regardless of NAT type.

---

### Gotcha #49 — Traefik v3 middleware cross-file references require `@file` suffix

**Area:** Traefik / Dynamic Config

**Symptom:** Service returns 404 despite Traefik router showing as enabled and
backend service running. No obvious error in router config.

**Root Cause:** In Traefik v3, middleware defined in a dynamic config file must
be referenced with the `@file` provider suffix when used in another file or in
stack labels. Without the suffix, Traefik cannot resolve the middleware and
silently drops it, causing requests to fall through with no middleware applied.

**Fix:** Update all middleware references from:
```
traefik.http.routers.<name>.middlewares=internal-only
```
to:
```
traefik.http.routers.<name>.middlewares=internal-only@file
```

**Applies to:** Any middleware defined in `/etc/traefik/dynamic/*.yml` and
referenced in Swarm service labels or other dynamic config files.

---

### Gotcha #50 — ZFS snapshot script silently failed for 20 days (wrong pool name)

**Area:** Backup / ZFS

**Symptom:** ZFS send cron job runs nightly without error in cron logs, but no
snapshots appear on the destination pool. `zpool list` shows only the expected
pools — no destination pool with the wrong name is created.

**Root Cause:** Script deployed 2026-04-10 with `DSTPOOL="oldpool"`. Correct pool
is `MainStorage`. `zfs receive` from a pipe fails silently when the destination
pool doesn't exist — no error propagates to the calling script unless
`set -euo pipefail` is active. No log file existed because the cron redirect
also failed silently. Discovered 2026-04-30 via `zpool list` — oldpool not present.

**Fix:** Always verify ZFS send targets exist before deploying cron jobs:
```bash
zpool list <target>
zfs list <target/dataset>
# Then do a manual test run and confirm the log file is created
```

**Rule:** Script fixed in `proxmox-swarm/scripts/zfs-snapshot.sh` — correct pool,
incremental sends, `set -euo pipefail`, per-dataset logging.

---

### Gotcha #51 — Synology NFS rejects NFSv4 for Docker volume mounts

**Area:** Backup / NFS / Synology

**Symptom:** `stack-restic.yml` deploys but restic-rest-server container immediately exits with NFS mount error. `docker volume inspect` shows volume created but mount fails.

**Root Cause:** Synology DSM does not reliably support NFSv4 for the Docker NFS volume driver with the options used (`addr`, `device`). The mount handshake fails silently at the volume level.

**Fix:** Set `nfsvers=3` explicitly in the Docker volume driver options:
```yaml
volumes:
  restic-data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.100.20,nfsvers=3,soft"
      device: ":/volume1/docker-backups"
```

---

### Gotcha #52 — Docker NFS volume retains stale mount options after stack redeploy

**Area:** Docker Swarm / Volumes

**Symptom:** After correcting NFS options in the stack file (Gotcha #51), the redeployed stack still fails with the old mount error. `docker volume inspect` shows the old `nfsvers=4` options still present.

**Root Cause:** Docker does not update an existing named volume's driver options on stack redeploy. The volume must be explicitly removed for the new options to take effect.

**Fix:** Force-remove the volume before redeploying:
```bash
docker stack rm restic
docker volume rm restic_restic-data   # exact name: <stack>_<volume>
docker stack deploy -c stack-restic.yml restic
```

---

### Gotcha #53 — Proxmox host cannot reach restic-rest-server on Swarm VLAN

**Area:** Networking / pfSense / Backup

**Symptom:** `restic-backup.sh` runs on the Proxmox host (`10.0.90.50`) and pushes to the restic-rest-server listening on port 8000 via the Swarm routing mesh (VLAN 60). Connection times out — `curl http://10.0.60.30:8000` from Proxmox host hangs.

**Root Cause:** pfSense had no rule permitting traffic from the Proxmox management VLAN (`10.0.90.0/24`) to VLAN 60 services. The restic-rest-server is only reachable via the routing mesh which publishes on all Swarm node IPs on VLAN 60.

**Fix:** Add pfSense firewall rule: source `10.0.90.50` → destination `10.0.60.0/24`, port `8000`, protocol TCP.

---

### Gotcha #54 — Portainer agent on a Swarm worker always runs in cluster mode

**Area:** Portainer / Docker Swarm

**Symptom:** `portainer/agent` deployed as a plain `docker run` on a Swarm worker node starts successfully but crashes in a loop. Logs show `running in cluster mode` followed by `unable to retrieve a list of IP associated to the host | error="lookup tasks. on <dns>: no such host"`. Setting `AGENT_CLUSTER_ADDR=""` suppresses the DNS crash but the agent then fails with `unable to redirect request to a manager node: no manager node found` when Portainer queries `/info`.

**Root Cause:** The Portainer agent detects Swarm mode from the Docker socket and forces cluster mode regardless of `AGENT_CLUSTER_ADDR`. In cluster mode it requires a Swarm manager to proxy certain API calls — a plain `docker run` container on a worker has no path to one outside the overlay network.

**Fix:** Use `tecnativa/docker-socket-proxy` instead of the Portainer agent, and register the environment as **Docker API** (not Agent) in Portainer:

```bash
docker run -d \
  --name docker-socket-proxy \
  --restart=always \
  -p 2375:2375 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e CONTAINERS=1 -e INFO=1 -e IMAGES=1 \
  -e VOLUMES=1 -e NETWORKS=1 -e POST=1 \
  tecnativa/docker-socket-proxy
```

In Portainer: **Add environment → Docker Standalone → Docker API** → address `<vm-ip>:2375`, TLS disabled.

**Security:** Port 2375 is unauthenticated. Lock the pfSense source rule to the Portainer host IP only (not the whole management VLAN subnet).

**Limitation:** Portainer cannot deploy Compose stacks through this environment — it detects Swarm mode and attempts Swarm stack deployment, which fails. Use this environment for container visibility, log access, and start/stop only. Manage `compose-vpn.yml` directly via `docker compose` on the VM.

---

### Gotcha #55 — Uptime Kuma monitors should use internal Swarm DNS, not Traefik hostnames

**Area:** Uptime Kuma / Monitoring

**Symptom:** Uptime Kuma monitors for internal services (Prometheus, Grafana, etc.) fail when configured with their Traefik hostname (e.g. `https://prometheus.home.purvishome.com`). The request originates from Uptime Kuma's container IP on the `traefik-public` overlay (`10.200.x.x`), which is not in the Traefik `internal-only` IPAllowList — so Traefik returns 403/connection refused.

**Root Cause:** The `internal-only` middleware allowlist covers admin client IPs (VLAN 60, Tailscale range), not internal overlay network ranges. Uptime Kuma is inside the Swarm — it doesn't need to go via Traefik to reach other services.

**Fix:** Use internal Swarm service DNS for all monitors on services in the same Swarm:

| Service | Wrong (Traefik URL) | Correct (Swarm DNS) |
|---|---|---|
| Prometheus | `https://prometheus.home.purvishome.com` | `http://prometheus:9090` |
| Grafana | `https://grafana.home.purvishome.com` | `http://grafana:3000` |
| Portainer | `https://portainer.home.purvishome.com` | `http://portainer:9000` |
| Sonarr | `https://sonarr.home.purvishome.com` | `http://sonarr:8989` |
| Radarr | `https://radarr.home.purvishome.com` | `http://radarr:7878` |
| Prowlarr | `https://prowlarr.home.purvishome.com` | `http://prowlarr:9696` |

**Exception — Transmission:** Transmission runs behind Gluetun (VPN sidecar) on `worker-mediamanagement-01` as Docker Compose, not Swarm. Swarm DNS does not resolve it. Use the Traefik hostname or monitor the Gluetun health endpoint instead.

**Rule:** For all Swarm-native services, always use `http://<service-name>:<port>` in Uptime Kuma. Only use Traefik hostnames for external services or Compose stacks outside Swarm.

---

### Gotcha #56 — Services missing from `traefik-public` network cause 502 Bad Gateway

**Area:** Traefik / Docker Swarm networking

**Symptom:** Browsing to a service's Traefik hostname returns 502 Bad Gateway. The Traefik dashboard shows the backend as unhealthy or unreachable.

**Root Cause:** Traefik routes traffic via the `traefik-public` overlay network. If a service has Traefik labels but is not attached to `traefik-public`, Traefik cannot reach the container — even though the service is running and healthy.

**Example:** InfluxDB had correct Traefik labels in `stack-monitoring.yml` but was only attached to the internal `monitoring` overlay. Adding `traefik-public` to the service's `networks` list fixed the 502.

**Fix:** Ensure every service with `traefik.enable=true` has `traefik-public` in its `networks` list:

```yaml
services:
  myservice:
    networks:
      - mystack_internal
      - traefik-public   # required for Traefik routing

networks:
  mystack_internal:
  traefik-public:
    external: true
```

**Rule:** Traefik labels without `traefik-public` network attachment = guaranteed 502.

---

### Gotcha #57 — unifi-poller v2: `UP_UNIFI_DEFAULT_PASS_FILE` env var not supported; Portainer secret UI adds trailing newline

**Area:** unifi-poller / Docker Swarm secrets

**Symptoms:**
- unpoller logs show continuous `status: 400: authentication failed` against the UniFi controller
- `curl` test of credentials directly against the UniFi API succeeds
- `docker exec <container> env | grep PASS` returns nothing (distroless image, no shell)

**Root Cause — two compounding issues:**

1. **`UP_UNIFI_DEFAULT_PASS_FILE` is not a supported environment variable in unpoller v2.** The env var approach works for `UP_UNIFI_DEFAULT_PASS` (plain password) but the `_FILE` suffix convention is not implemented. The only supported way to reference a secret file is via the TOML config key `pass_file`.

2. **Portainer's secret creation UI adds a trailing newline.** When you type a secret value in the Portainer UI and click Create, the stored secret content is `password\n`. If unpoller reads this via `pass_file`, it sends `password\n` as the credential, which the UniFi controller rejects with 400. Creating secrets via the Portainer UI is not safe for file-based password consumption.

**Fix — volume-mounted TOML config with inline password:**

The simplest reliable approach for a homelab is to put the password directly in the TOML config file, mounted from the Proxmox host via virtiofs:

1. Create `/mnt/docker-data/unpoller/up.conf` on the Proxmox host:

```toml
[global]
  quiet = false
  debug = false

[prometheus]
  disable = false
  http_listen = "0.0.0.0:9130"
  namespace = "unpoller"

[influxdb]
  disable = true

[unifi]
  dynamic = false

[[unifi.controller]]
  url        = "https://unifi:8443"
  user       = "unpoller"
  pass       = "YOURPASSWORD"
  verify_ssl = false
  save_sites = true
```

2. Mount it in the stack — no `secrets:` or `configs:` needed:

```yaml
  unpoller:
    image: ghcr.io/unpoller/unpoller:latest
    volumes:
      - /mnt/docker-data/unpoller/up.conf:/etc/unpoller/up.conf:ro
```

**If Docker secrets are required** (e.g. for `pass_file`), always create them via CLI using `printf` to avoid trailing newlines:

```bash
printf 'YOURPASSWORD' > /tmp/pass
docker secret create unpoller_password /tmp/pass
rm /tmp/pass
```

Never use `echo` (adds `\n`) or the Portainer UI for secrets that will be consumed as raw file content.

**UniFi account requirements:** The `unpoller` user must be a local admin (not a cloud account) with **Super Admin** role. Create it in UniFi → Settings → Admins → Add Admin → set role to Super Admin and disable "Allow this admin to be managed by Ubiquiti SSO".

---

### Gotcha #58 — Plain-compose stacks attaching Swarm overlays fail to auto-start at boot

**Area:** Docker Compose / Docker Swarm / systemd

**Symptom:** A `docker-compose.yml` stack with `restart: unless-stopped` whose services attach a Swarm-managed overlay network (e.g. `arr_arr_default`, `traefik-public`) does not come back up after a VM reboot. `docker logs` on the failed container shows `failed to attach to network <overlay>: context deadline exceeded`. Manual `docker compose up -d` minutes later succeeds.

**Root Cause — two-layer race on boot:**

1. **Local visibility race.** `docker network inspect <overlay>` reports the network exists locally before Swarm gossip has finished propagating it from the manager node. Polling network existence is not a reliable readiness signal.
2. **Attach race.** Even after `inspect` succeeds, the *first* container attach can still fail with `context deadline exceeded` until manager-node gossip fully completes. Docker's `restart: unless-stopped` retries are spaced too aggressively to outlast this — the daemon hits its retry cap and gives up before gossip is ready.

The only reliable readiness signal is a transient probe container actually attaching to each overlay successfully.

**Fix — systemd oneshot unit with probe-based readiness + retry loop.** New file `proxmox-swarm/systemd/compose-vpn.service`:

- `Requires=docker.service`, `After=docker.service network-online.target`
- `ExecStartPre`: wait for `Swarm.LocalNodeState=active`, both overlays inspectable, AND a transient `alpine` container with `--network <overlay>` actually starts (probe attach). Loop with timeout.
- `ExecStart`: `docker compose -p compose-vpn -f /opt/compose-vpn/compose-vpn.yml up -d`, retried up to 5× with `down --remove-orphans` between attempts.
- `-p compose-vpn` pins project name (prevents container-name conflict if stack was originally started from a different cwd).

**Provisioning generalization (so future plain-compose stacks slot in the same way):**

- `02-provision-vm.sh` accepts repeatable `--compose-unit <name>` flag → writes `compose_units` array to VM registry JSON.
- `03-post-boot.sh` → new `install_compose_units` step copies `/mnt/docker-swarm/stacks/<name>.yml` → `/opt/<name>/<name>.yml` and `/mnt/docker-swarm/systemd/<name>.service` → `/etc/systemd/system/<name>.service`, then `systemctl enable` (does *not* auto-start — env files / secrets must be in place first).
- `copy-swarm-config.sh` syncs `proxmox-swarm/systemd/*.service` → `/mnt/docker-swarm/systemd/` so unit files reach all VMs via virtiofs.

**Replay on existing VM:**

```bash
sudo install -m 644 /mnt/docker-swarm/systemd/compose-vpn.service /etc/systemd/system/compose-vpn.service
sudo systemctl daemon-reload
sudo systemctl enable --now compose-vpn.service
```

**Rule:** Any plain-compose stack on a Swarm node that attaches a Swarm overlay must use a systemd oneshot unit with probe-based overlay readiness. `restart: unless-stopped` alone is insufficient.

**Reference:** PR [#25](https://github.com/scarredNinja/docker-swarm-home/pull/25), branch `claude/mystifying-engelbart-998246`.

---

### Gotcha #59 — `cloudflare/cloudflared:latest` is distroless; tunnel token must be persisted in Portainer stack env vars

**Area:** cloudflared / Plex external access / Portainer / Docker Swarm

**Symptom:** `plex_cloudflared` service stuck at 0/1 replicas with `Update paused due to failure`. `docker service inspect plex_cloudflared` shows `Args: tunnel --no-autoupdate run --token` — the token argument is **empty**. Container exit code 255. No errors at deploy time.

**Root Cause — env-var interpolation source-of-truth was the operator's shell:**

`stack-plex.yml` had:

```yaml
command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
```

`docker stack deploy` interpolates `${CLOUDFLARE_TUNNEL_TOKEN}` from the **deploying operator's shell**, not from any persistent store. The token was never persisted on the manager — fresh ssh sessions, reboots, or Portainer redeploys without the env var would silently produce a broken stack with `--token ""`.

**Failed attempt — Docker secret + entrypoint shim (PRs #26, #27, reverted in #28):**

Tried:

```yaml
entrypoint: ["sh", "-c", 'exec cloudflared tunnel --no-autoupdate run --token "$$(cat /run/secrets/cf_tunnel_token)"']
```

Container failed to start with: `exec: "/bin/sh": stat /bin/sh: no such file or directory`.

**Why it failed:** `cloudflare/cloudflared:latest` is **distroless** — no `/bin/sh`, no busybox, no debug tools. The image contains only the `cloudflared` binary and its runtime dependencies. Entrypoint shims of any kind are impossible.

**Fix — persist the token in Portainer's stack environment variables:**

1. Keep the original `command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}` form in `stack-plex.yml`.
2. In Portainer: Stacks → plex → Environment variables → add `CLOUDFLARE_TUNNEL_TOKEN=<token>`.
3. Redeploy.

Portainer stores stack env vars in its encrypted DB and always interpolates at deploy time, regardless of which shell or workflow triggered the deploy. Survives manager reboots, fresh ssh sessions, and redeploys from Git webhook / UI / API equally.

> [!note] PR #61 update (2026-05-26) — token now stored as Docker secret
> The Portainer env-var approach above has been superseded. In PR #61 the tunnel token was moved to a Docker secret `cloudflare_tunnel_token`. The container receives it via the `TUNNEL_TOKEN` environment variable set from the secret at startup. Create the secret with:
> ```bash
> printf '%s' '<token>' | docker secret create cloudflare_tunnel_token -
> ```

**If a wrapper around `cloudflared` is ever required** (e.g. to read a secret from `/run/secrets/...`), the only path is to build a thin custom image based on `alpine` or `busybox` that copies the cloudflared binary in. The official image cannot be shimmed.

**Two ways the env-var-interpolation pattern can still fail in the future:**
1. Stack deployed via `docker stack deploy` from a shell that lacks the var → silent empty interpolation.
2. Portainer stack env var deleted → same silent failure.

**Mitigation (not yet implemented):** add a healthcheck that fails fast on a missing or empty token to surface the issue immediately.

**Reference:** PRs [#26](https://github.com/scarredNinja/docker-swarm-home/pull/26), [#27](https://github.com/scarredNinja/docker-swarm-home/pull/27) (superseded), [#28](https://github.com/scarredNinja/docker-swarm-home/pull/28) (revert + landed solution). Commits `8917230`, `5b87840`, `c6f6318`.

---

### Gotcha #60 — Bind-mounting the parent of autofs triggers gives empty containers

**Area:** Docker bind mounts / autofs / NFS / Plex

**Symptom:** Plex container saw empty `/media/Movies`, `/media/TVShows`, `/media/Animation` despite the NFS shares being mounted on the host at `/mnt/media/*`. Libraries had no content visible from inside the container, yet `ls /mnt/media/Movies` on the host worked fine.

**Root Cause — Docker's default bind propagation (`rprivate`) does not see autofs-triggered submounts:**

The host has each NFS share mounted via **autofs direct-mount triggers** at `/mnt/media/{Movies,TVShows,Animation}`, sourced from `10.0.100.20:/volume1/<share>`. The Plex stack bind-mounted the parent:

```yaml
- /mnt/media:/media:ro
```

When Docker resolves a bind mount with default `rprivate` propagation, it captures the source path's namespace at mount time — including the autofs trigger directories themselves, but NOT the underlying NFS mounts that those triggers fire when accessed. The container sees the trigger directories as empty.

**Fix — bind each share directly:**

```yaml
- /mnt/media/Movies:/media/Movies:ro
- /mnt/media/TVShows:/media/TVShows:ro
- /mnt/media/Animation:/media/Animation:ro
```

Each source is itself the autofs-triggered mount, so Docker captures the live NFS data. Container-side paths under `/media/<share>` are unchanged → no Plex library reconfiguration needed.

This pattern was already correct in [stack-arr.yml](proxmox-swarm/stacks/stack-arr.yml) — the plex stack predated it.

**Rule:** Whenever a bind-mount source could contain dynamically-triggered mounts (autofs, systemd automount units, etc.), bind the trigger directly — never its parent. Default Docker bind propagation does not follow triggers.

**Alternative (not used):** `bind: propagation: rshared` would make the container see new submounts as they appear, but requires the source to be a shared mount on the host and is more invasive than just binding each share directly.

**Reference:** PR [#30](https://github.com/scarredNinja/docker-swarm-home/pull/30), commit `ffd7d90`.

---

### Gotcha #61 — vm-backup.sh silent failure under cron (PATH)

**Area:** Backup / Cron

**Symptom:** `vm-backup.sh` ran under cron with no output, no Discord alert,
no Prometheus metrics written. No vzdump backups taken. Script worked
correctly when run manually as root.

**Root Cause:** Cron inherits `PATH=/usr/bin:/bin` only. Proxmox tools `qm`,
`vzdump`, `pvesm` live in `/usr/sbin` — not in cron's minimal PATH.

**Fix:** Pin `PATH=/usr/sbin:/usr/bin:/sbin:/bin` at the top of any script
that calls Proxmox CLI tools. Add an ERR trap so future failures alert and
write metrics rather than silently exit.

**Rule:** Always run scripts manually first, then verify under cron separately.
`sudo env -i PATH=/usr/bin:/bin bash script.sh` simulates cron's environment.

---

### Gotcha #62 — Restic forget/prune blocked by stale lock from dead PID

**Area:** Backup / Restic

**Symptom:** `restic backup` succeeds nightly but `restic forget` and
`restic prune` silently skip. Repo grows unbounded — old snapshots not pruned.
No error in log unless explicitly checked.

**Root Cause:** A previous restic process died mid-operation (Synology 500
error, 2026-05-08), leaving a lock file in the repo with a now-dead PID
(847046). Every subsequent night restic found the lock and aborted forget/prune
as a safety measure. Backup itself is unaffected by locks.

**Fix:** Add `restic unlock` before the forget/prune step. Restic's unlock
only removes locks whose owner PID is confirmed dead — safe to run nightly.
Will not clear a lock from a live concurrent process.

**Check command:**

```bash
restic locks  # lists all active locks with PID and hostname
```



---

### Gotcha #63 — Plex DNS resolving to Traefik instead of direct Plex IP

**Area:** Plex / Networking / Pi-hole

**Symptom:** Internal Plex streaming slow and stuttering despite correct
pfSense rules. DNS for `plex.purvishome.com` returned `10.0.60.40` (Traefik's
VLAN 60 interface) instead of `10.0.50.20` (Plex).

**Root Cause:** Pi-hole local DNS entry was pointing at Traefik. All media
traffic routed through an unnecessary HTTP proxy — two pfSense hops instead
of one direct connection.

**Fix:** Pi-hole → Local DNS → DNS Records:
- `plex.purvishome.com` → `10.0.50.20`
- `plex.home.purvishome.com` → `10.0.50.20`

pfSense rule required: VLAN10 net → `10.0.50.20` TCP 32400.

**Rule:** Plex must never route through Traefik. Plex handles its own TLS
via `*.plex.direct` certificates on port 32400. Set Pi-hole DNS to the
direct Plex container IP for all Plex hostnames.

---

### Gotcha #64 — VM CPU type `kvm64` degrading transcode performance

**Area:** Proxmox / VM / Plex

**Symptom:** Software transcode performance far below equivalent LXC setup.
ffmpeg unable to sustain 1080p H264 transcode in real-time (Speed: 0.2).
Same content transcoded without issue on previous LXC-based Plex.

**Root Cause:** `cpu: kvm64` is a lowest-common-denominator vCPU that strips
modern instruction sets including AVX2 and SSE4.2. ffmpeg uses these
extensively for video encoding via SIMD optimisation. LXC containers bypass
virtualisation entirely and see the full host CPU.

**Fix:**

```bash
qm set <vmid> --cpu host
# Requires VM reboot to take effect
qm config <vmid> | grep cpu  # verify before and after
```


**Rule:** Always set `--cpu host` on VMs running CPU-intensive workloads
(transcoding, database engines, compression). Only use `kvm64` when live
migration across heterogeneous CPU hosts is required.

---

### Gotcha #65 — `curl --fail` exits non-zero on HTTP 405 from restic REST server

**Area:** Backup / restic

**Symptom:** Every nightly restic backup aborts before reading a single file. Log shows preflight check failure immediately after `START`. No files are backed up.

**Root Cause:** The restic REST server returns HTTP 405 Method Not Allowed on a plain `GET /` request — this is correct behaviour, it only accepts POST for API calls. `curl --fail` treats any non-2xx response as a failure and exits non-zero, which caused the script to abort via `set -e`.

**Fix:** Remove `--fail` from the preflight curl. A successful TCP connection (exit 0 regardless of HTTP status) is sufficient to confirm the server is up.

```bash
# Wrong — exits non-zero on HTTP 405
curl --fail --silent --max-time 5 "$RESTIC_REST_URL" > /dev/null

# Correct — succeeds on any HTTP response, fails only on connection error
curl --silent --max-time 5 "$RESTIC_REST_URL" > /dev/null
```

---

### Gotcha #66 — `qm` not found in cron — `/usr/sbin` absent from cron PATH

**Area:** Backup / cron / vzdump

**Symptom:** `vm-backup.sh` logs `START` then exits silently. All VMs appear to be skipped. No Discord alert fires (the error occurs before the alert logic runs).

**Root Cause:** Cron's default PATH is typically `/usr/bin:/bin`. `qm` (the Proxmox VM management binary) lives in `/usr/sbin`, which is not included. The `qm` command silently fails, the script exits 0 (or the loop continues without error), and no backups are taken.

**Fix:** Add an explicit PATH export at the top of any script that calls Proxmox or system binaries (`qm`, `vzdump`, `pvesm`, `zfs`, `zpool`):

```bash
export PATH=/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin
```

This must be the first non-comment line after the shebang. Do not rely on the caller's PATH — cron, systemd timers, and SSH non-login shells all have minimal environments.

---

### Gotcha #67 — Restic stale lock causes all subsequent nightly runs to abort

**Area:** Backup / Restic

**Symptom:** Nightly restic backup exits immediately after START with
"unable to create lock in backend: repository is already locked by PID <n>".
PID no longer exists. All subsequent nights fail the same way with no files
backed up.

**Root Cause:** When restic fails or is killed mid-run, it cannot clean up
its repository lock. Every subsequent run attempts to create a new lock,
finds the stale one, and aborts.

**Fix:** Add `restic unlock --remove-all` before every backup run in
restic-backup.sh. No-op on a clean repo; auto-heals a stale lock.

To manually clear an existing stale lock:

```bash
restic -r rest:http://10.0.60.40:8000/ \
  --password-file /etc/restic-password \
  unlock --remove-all
```

**Rule:** Any long-running restic operation that can be interrupted (backup,
forget, prune) can leave a stale lock. Always unlock at script startup.

---

### Gotcha #68 — restic-rest-server NFS `soft,timeo=30` crashes container under write load

**Area:** Backup / Docker Swarm / NFS

**Symptom:** restic backup fails mid-transfer with `connection reset by peer`
then `500 Internal Server Error` then `connection timed out`. Service log
shows: `sync /data/...: input/output error`. Swarm reschedules the container
multiple times.

**Root Cause:** Docker volume NFS mount options `soft,timeo=30` (3-second
timeout). Under heavy write load (100+ GiB), any NFS hiccup >3s returns
EIO to the container process. The rest-server gets input/output error on
fsync and exits. Swarm reschedules but the backup client has already lost
connection.

**Fix:** Change NFS volume driver options in stack-restic.yml:

```yaml
# Before
o: addr=10.0.100.20,nfsvers=3,rw,soft,timeo=30

# After
o: addr=10.0.100.20,nfsvers=3,rw,hard,timeo=600,retrans=5
```

Requires volume removal and stack redeploy (data is safe — lives on NAS):

```bash
docker stack rm restic
docker volume rm restic_restic-data
docker stack deploy -c stack-restic.yml restic
```

**Rule:** Never use `soft` NFS mounts for write-heavy workloads or services
that cannot tolerate EIO. Use `hard` with a generous `timeo`. `soft` is
only appropriate for read-only or latency-sensitive mounts where a hung
process is worse than a failed write.

> [!note] 2026-05-22 update — PR #50 reverted `hard,timeo=600,retrans=5` back to `soft,timeo=50,retrans=3`. The `hard` mount caused Go goroutine starvation under backup load (Gotcha #49): goroutines parked indefinitely on NFS stall with no recovery path. The correct tradeoff at homelab scale is soft with a short timeout — backup EIO is survivable; hung goroutines are not. See Gotcha #49 for the full rationale.

---

### Gotcha #69 — nfs-bench.sh metrics not appearing in Grafana despite .prom file present

**Area:** Monitoring / Prometheus / node-exporter

**Symptom:** .prom file confirmed on worker-media-01 at
/var/lib/node_exporter/textfile_collector/. node-exporter serving it.
Grafana panels show "No data".

**Root Cause (suspected):** Prometheus scrape config does not include
10.0.50.50:9100 (worker-media-01 node-exporter). Metrics never ingested.

**Diagnostic:**

```bash
docker exec $(docker ps -q --filter name=monitoring_prometheus) \
  wget -qO- 'http://localhost:9090/api/v1/targets' | python3 -c "
import json,sys
data = json.load(sys.stdin)
for t in data['data']['activeTargets']:
    if 'node' in t['labels'].get('job',''):
        print(t['labels'].get('instance'), t['health'], t.get('lastError','')[:80])
"
```

**Fix:** If 10.0.50.50:9100 missing from targets, add to prometheus.yml
scrape config under the node-exporter job and redeploy monitoring stack.

**Status:** Fixed in [PR #40](https://github.com/scarredNinja/docker-swarm-home/pull/40) — 2026-05-18.

---

### Gotcha #70 — OpenVPN single-threaded core pin causes UI latency on Arr stack

**Area:** Gluetun / VPN / worker-mediamanagement-01 / Performance

**Symptom:** Radarr/Sonarr UI takes 3–8s to load despite container CPU <1%.
vCPU bump from 4→6 (2026-05-08) reduced CPU spike frequency but did not
resolve root cause. `top` on the VM shows one gluetun thread pinned at ~100%
of a single vCPU during active torrenting.

**Root Cause:** OpenVPN is single-threaded by design. All encryption/decryption
runs on a single core regardless of how many vCPUs are available. At download
saturation, this pins one vCPU to 100%, causing the hypervisor to schedule
competing processes (Sonarr/Radarr DB queries, UI request handling) on the
same physical core — manifesting as UI latency even though the Arr containers
themselves appear idle.

**Fix:** Migrate Gluetun from OpenVPN to WireGuard (NordLynx protocol on NordVPN).
WireGuard is multi-threaded, kernel-native, and benchmarks at approximately 5×
the CPU efficiency of OpenVPN for equivalent throughput. Expected result: gluetun
CPU drops below 10% at full download saturation, freeing all vCPUs for Arr workloads.

**Migration steps (compose-vpn.yml):**

1. Extract WireGuard private key from NordVPN client on any device with the client:

```bash
nordvpn set technology nordlynx
nordvpn connect
wg show wg0 private-key
```

   Copy the key — it is stable per account (not per session).

2. In compose-vpn.yml, replace OpenVPN env vars with:

```yaml
- VPN_TYPE=wireguard
- VPN_SERVICE_PROVIDER=nordvpn
- WIREGUARD_PRIVATE_KEY=<key from step 1>
- SERVER_COUNTRIES=New Zealand
- SERVER_CATEGORIES=P2P
- FIREWALL_OUTBOUND_SUBNETS=10.0.0.0/8
```

3. On worker-mediamanagement-01:

```bash
cd /opt/compose-vpn
docker compose down
docker compose up -d
docker compose logs gluetun | tail -30
# Look for: "Connected" and a WireGuard endpoint line
```

4. Verify tunnel is live:

```bash
# Should show a NordVPN IP, not your real WAN IP
docker exec compose-vpn-gluetun-1 curl -s https://ifconfig.me
```

**Rule:** Always use WireGuard over OpenVPN for sidecar VPN containers. OpenVPN's
single-threaded encryption model is a performance liability on multi-workload VMs.
The WireGuard private key for NordVPN is account-stable — extract once, store
in compose-vpn.yml (which is on the virtiofs share, not in the public git repo).

---

### Gotcha #71 — Restic deferred retry: lock must be released before calling restic from vm-backup.sh

**Area:** Backup / Cron / Restic

**Symptom:** `restic-backup.sh` defers when vm-backup is running (correct
behaviour — both scripts use a lock to prevent concurrent runs). But if
`vm-backup.sh` triggers restic directly at the end of its run without
releasing its own lock first, restic detects the vm-backup lock still held,
defers again, writes a new sentinel, and exits. The chain never completes.

**Root Cause:** The lock release and EXIT trap disarm in `vm-backup.sh` were
not happening before the restic call. The sequence was:

    vm-backup.sh finishes work
    → calls restic-backup.sh   # lock still held
    → restic sees lock, defers, writes new sentinel, exits
    → sentinel remains, restic never runs

**Fix:** In `vm-backup.sh`, explicitly release the lock and disarm the EXIT
trap before calling `restic-backup.sh`:

```bash
rm -f /tmp/vm-backup.lock   # release lock
trap - EXIT                 # disarm so trap doesn't re-acquire on exit
/opt/scripts/restic-backup.sh   # now safe — lock is clear
```

**Rule:** Any script that chains into another lock-aware script must release
its own lock first. "Release → trigger" is the correct order, not
"trigger → release." The 08:00 safety-net cron added by
`deploy-backup-scripts.sh` acts as a backstop if `vm-backup.sh` crashes
before reaching the trigger step.

---

### Gotcha #72 — Prometheus DNS SD misses VLAN 50 media VM node-exporters

**Area:** Monitoring / Prometheus / worker-media-01 / worker-mediamanagement-01

**Symptom:** `nfs_bench.prom` present on `worker-media-01`, node-exporter serving it
(`curl http://10.0.50.50:9100/metrics | grep nfs_bench` returns results). Grafana
panels show "No data". `10.0.50.50:9100` is absent from Prometheus → Status → Targets.

**Root Cause:** Prometheus job `node-exporter` uses `dns_sd_configs` with
`tasks.monitoring_node-exporter`. This resolves only Swarm task IPs on the monitoring
overlay network. `worker-media-01` (10.0.50.50) and `worker-mediamanagement-01`
(10.0.50.51) run node-exporter as native systemd services on VLAN 50. They are not
Swarm tasks and are never discovered by DNS SD.

**Fix:** Added `node_exporter_external` job in `config/monitoring/prometheus.yml` with
`static_configs` targeting `10.0.50.50:9100` and `10.0.50.51:9100`.

Also requires pfSense rule (manual): Interface VLAN60, Pass, source `10.0.60.0/24` →
destination `10.0.50.0/24`, TCP port 9100. Without this rule, Prometheus cannot reach
the VLAN 50 node-exporters even with the correct scrape config.

**Status:** Fixed in [PR #40](https://github.com/scarredNinja/docker-swarm-home/pull/40) — 2026-05-18.

---

### Gotcha #73 — nfs-bench.sh: missing nfs_bench_last_exit_code metric

**Area:** Monitoring / nfs-bench / Grafana alerting / worker-media-01

**Symptom:** No way to alert when `nfs-bench.sh` itself fails mid-run. If the script
errors out (e.g. NFS mount gone, fio crash), the `.prom` file either retains stale
values from the previous run or is absent — Grafana shows old data with no indication
of failure.

**Root Cause:** The original script had no metric tracking whether the last run
succeeded or failed. The `err_exit` trap just called `exit 1` with no .prom update.

**Fix:** Added `nfs_bench_last_exit_code{host="..."}` gauge:
- Written as `0` (atomically via tmp→mv) when the script completes successfully
- Written as `1` by `err_exit()` before exiting on any ERR trap

**Status:** Fixed in [PR #40](https://github.com/scarredNinja/docker-swarm-home/pull/40) — 2026-05-18.

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
  $(docker node ls --filter name=worker-mediamanagement-01 -q)
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


## Appendix F

### Know Issues

- [x] `traefik.yml` — `delayBeforeChecks` fixed to `delayBeforeCheck: "10s"` directly under `dnsChallenge` ✅ 2026-04-18
- [x]  `traefik.yml` — encoding artifact fixed ✅ 2026-04-18
- [x]  `stack-monitoring.yml` — pin `influxdb:2.7` (still pending — add to code session)
- [x] Traefik cert resolver not loading — container may not be reading updated `traefik.yaml`. Verify mount path and confirm `delayBeforeCheck` present inside running container. [priority:: 1] ✅ 2026-04-23
- [x] `monitoring_monitoring` network not found — deploy `stack-monitoring` before `stack-traefik`, or create network manually before Traefik redeploy priority:: 1] ✅ 2026-04-23
- [x] Confirm `monitoring` overlay network joined to Traefik stack (not just IP scraping) priority:: 1] ✅ 2026-04-23
- [x] `traefik-dmz-01` — permanent asymmetric routing fix: add `dhcp4-overrides: use-routes: false` to `enp6s19` stanza in netplan. Temp `ip route del` applied 2026-04-21 but does not survive reboot. [priority:: 1] ✅ 2026-04-29 (resolved by NIC binding)
- [x] `zpool export MainStorage` — must run before any Proxmox reboot [priority:: 2]
- [x] `traefik-dmz-01` — bind entrypoints to specific NICs: `websecure` → `10.0.80.20:443`, `internal` → `10.0.60.40:443`, add `web-internal` redirect on `10.0.60.40:80`. Update `infrastructure.yml` internal service labels from `websecure` → `internal`. Fix `60-nic2-dmz.yaml` netplan `use-routes: false` permanently. [priority:: 1] ✅ 2026-04-29
- [x] unifi.home.purvishome.com returns 404 — Traefik logs pending [priority:: 1] ✅ 2026-04-30
- [x] Sonarr/Radarr UI extremely slow to load after arr stack redeploy — Radarr UI stuck on loading screen for 5+ min, direct port access (10.0.50.51:7878) returns connection refused. Sonarr affected too. Resolved by migrating from OpenVPN to NordLynx WireGuard on 2026-05-17 and correcting Prometheus scrape target IPs on 2026-05-20. [priority:: 2] ✅ 2026-05-20

---

## Appendix G — Overlay Network Recovery

> [!abstract] Use this when Portainer agents are not visible, overlay-attached services cannot communicate, or `docker network inspect` shows nodes missing from peer list. Root cause is almost always stale VXLAN interfaces or a stale `local-kv.db` on one or more nodes — typically after an unclean reboot of `manager-01`.

### Symptoms

- Portainer UI shows agents as disconnected or shows only manager node
- `docker network inspect <overlay> --format '{{json .Peers}}'` shows nodes missing
- Services on different VMs cannot reach each other across an overlay network
- `docker service ps <service>` shows tasks stuck in `Preparing` or cycling

### Step 1 — Diagnose overlay peer state (manager-01)

```bash
# Check which nodes are visible as peers on each overlay
docker network ls --filter driver=overlay
docker network inspect traefik-public --format '{{json .Peers}}' | python3 -m json.tool

# All Swarm nodes should appear. Missing nodes = stale VXLAN or stale KV store.
```

> [!important] `docker network inspect` only shows containers local to the node you run it on for non-attachable overlays. To test actual overlay reachability between nodes, use nsenter — not a host-level ping:
> ```bash
> # Get the short network namespace ID (first 10 chars of network ID shown in docker network ls)
> docker network ls
>
> # Enter the network namespace on manager-01 and ping the overlay IP of the target container
> sudo nsenter --net=/var/run/docker/netns/1-<short-netid> ping <target-overlay-ip>
> ```
> A successful ping from inside the netns confirms the overlay is functioning end-to-end.

### Step 2 — Remove stale VXLAN interfaces (manager-01)

```bash
# List VXLAN interfaces — stale ones remain after Docker crash or unclean reboot
ip link show type vxlan

# Delete each stale interface (repeat for each listed)
sudo ip link delete <vxlan-interface-name>

# Restart Docker to rebuild overlay tables cleanly
sudo systemctl restart docker
```

### Step 3 — Clear stale local-kv.db (all nodes)

Run on each Swarm node (manager-01 first, then workers):

```bash
sudo systemctl stop docker
sudo rm -f /var/lib/docker/network/files/local-kv.db
sudo systemctl start docker
```

> [!warning] Do this on manager-01 first, then workers one at a time. Stopping Docker on a worker is safe — Swarm reschedules its tasks. Stopping Docker on the manager briefly interrupts the control plane but does not lose Swarm state (Raft is on virtiofs).

### Step 4 — Force re-registration of affected services

```bash
# Force-update any service that was on the broken overlay
# This triggers task replacement and a fresh overlay IP assignment
docker service update --force portainer_portainer_agent
docker service update --force <other-affected-service>
```

### Step 5 — Verify

```bash
# Peer list should now show all nodes
docker network inspect traefik-public --format '{{json .Peers}}' | python3 -m json.tool

# Services should reach each other
sudo nsenter --net=/var/run/docker/netns/1-<short-netid> ping <overlay-ip>

# Portainer agents should all appear in the UI
```

---

## Appendix H — Post-Reboot Checklist

> [!abstract] Run through this after any planned or unplanned reboot of `manager-01` or any worker VM. Most items take under a minute to verify. Catching issues here prevents silent failures that only surface when you try to use a service.

### After manager-01 reboot

```bash
# 1. Confirm Swarm is healthy
docker info | grep -A5 "Swarm"
# Expected: Swarm: active, Is Manager: true

# 2. Confirm all nodes are Ready
docker node ls
# All nodes should show Ready / Active

# 3. Confirm virtiofs mounts are present
ls /mnt/docker-data/ /mnt/docker-tsdb/ /mnt/docker-db/ /mnt/docker-swarm/
# If any are empty — virtiofs failed to mount. Check: systemctl status virtiofsd@*

# 4. Confirm overlay peer state (see Appendix G if nodes missing)
docker network inspect traefik-public --format '{{json .Peers}}' | python3 -m json.tool

# 5. Force-update Portainer agent to ensure overlay re-registration
docker service update --force portainer_portainer_agent

# 6. Check running services
docker service ls
# All services should show replicas matching desired (e.g. 1/1, not 0/1)
```

### After any worker VM reboot

```bash
# On the worker VM — confirm virtiofs mounts
ls /mnt/docker-data/ /mnt/docker-swarm/
# If empty: systemctl status virtiofsd@docker-data virtiofsd@docker-swarm

# On manager-01 — confirm worker re-joined and is Ready
docker node ls | grep <worker-name>

# Check services on that worker have restarted
docker service ps <stack>_<service> | head -5
# Most recent task should show Running
```

### Gluetun VPN tunnel verification (worker-mediamanagement-01)

Transmission + Gluetun run as Docker Compose on `worker-mediamanagement-01` with `restart: unless-stopped` — they auto-restart after VM reboot. No manual intervention needed.

To verify the tunnel is healthy after a reboot:

```bash
ssh ubuntu@worker-mediamanagement-01

# VPN tunnel up
docker logs gluetun 2>&1 | tail -10
# Look for: "Forwarded port" or "Connected" line

# Transmission responding
curl -s http://localhost:9091/transmission/rpc | head -3
# Expected: 409 Conflict
```

If containers are not running (e.g. after a `docker compose down`), restart with:

```bash
cd /mnt/docker-swarm/stacks
docker compose -f compose-vpn.yml up -d
```

### Traefik cert resolver check

```bash
# On Proxmox host — confirm acme.json permissions (must be 600)
stat -c "%a %n" /mnt/docker-data/traefik/data/acme.json
# Expected: 600

# If 644 or 755, fix and restart Traefik
chmod 600 /mnt/docker-data/traefik/data/acme.json
docker service update --force traefik_traefik
```