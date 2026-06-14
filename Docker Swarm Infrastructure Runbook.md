---
type: reference
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm'
tags:
  - docker-swarm
  - reference
last_updated: '2026-04-21T00:00:00.000Z'
status: Reference
---

# Docker Swarm Infrastructure Runbook
**Path:** Docker Swarm Infrastructure Runbook.md
**Modified:** 2026-06-01T23:20:32.316Z
**Tags:** DockerSwarm, Infrastructure, Runbook, backlog, docker, homelab, migration, proxmox, swarm, traefik
**Frontmatter:**
  - type: "reference"
  - project_id: "Homelab-2025"
  - phase: "Phase 5: Docker Swarm"
  - tags: ["DockerSwarm","Runbook","Infrastructure"]
  - last_updated: "2026-04-21T00:00:00.000Z"

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
| 1 | Reboot `worker-media-01` — verify `--cpu host` took effect, retest Plex transcode | ✅ Done | Verified CPU host virtualization stable ✅ 2026-06-02 | Set this session, reboot not yet done |
| 2 | Verify overnight backup cron — all 4 logs + Grafana dashboard | ⏳ Pending | Check after 05:15 NZST |
| 3 | Plex — H264 + AAC Direct Play retest after VM reboot | 🔲 Pending | Expect Direct Play, no transcode |
| 4 | `traefik-dmz-01` — permanent `use-routes: false` on `enp6s19` netplan | ✅ Done | Temp fix 2026-04-21, does not survive reboot. See Appendix F |
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
- [x] Confirm credential files absent from synced vault on dev-node-01 ✅ 2026-06-02

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

#### Phase 1 Rollback

[… truncated at 50000 chars]
