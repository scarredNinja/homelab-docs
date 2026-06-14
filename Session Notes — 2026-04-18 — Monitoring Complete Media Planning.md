---
date: '2026-04-18T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 5: Docker Swarm — Monitoring + Media'
tags:
  - SessionNotes
  - DockerSwarm
  - Monitoring
  - Traefik
  - Prometheus
  - Grafana
  - InfluxDB
  - Proxmox
  - Media
  - Plex
session_type: Planning + Code
status: Completed
---

# Session Notes — 2026-04-18

## 🎯 Session Goals

1. Wire Traefik metrics into Prometheus
2. Connect Grafana data sources (Prometheus, InfluxDB)
3. Import Proxmox dashboard
4. Begin media VM provisioning

---

## ✅ Completed

### Traefik → Prometheus Metrics

Enabled Prometheus metrics in `traefik.yml` static config:

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addServicesLabels: true
    addRoutersLabels: true
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0
```

- Metrics exposed on port `:8080` (dedicated `:8082` entrypoint deferred to later code session)
- `traefik_traefik:8080/metrics` scrape confirmed working in Prometheus
- Grafana dashboard imported: **ID 17346** (Traefik v3 official) — showing live data

**traefik.yml validation findings:**

| Issue | Detail | Fix |
|-------|--------|-----|
| `delayBeforeChecks: 10` | Bare int = 10 nanoseconds, not 10 seconds | Change to `"10s"` (quoted string) — **deferred to code session** |
| Comment encoding artifact | `â€"` in inline comment | Fix to `—` — **deferred to code session** |

> [!bug] Deferred — traefik.yml fixes
> `delayBeforeChecks: "10s"` and comment encoding fix not yet applied. Add to next traefik code session.

---

### Grafana — Prometheus Data Source

- Connected: `http://monitoring_prometheus:9090`
- Save & Test: ✅ passing
- Dashboards imported:

| ID | Name | Data Source |
|----|------|-------------|
| 17346 | Traefik v3 Official | Prometheus |

---

### Grafana — InfluxDB Data Source

- Connected: `http://monitoring_influxdb:8086`
- Query language: **Flux**
- Org: `home`, Bucket: `proxmox_metrics`
- Save & Test: ✅ passing

### Proxmox → InfluxDB Push

Configured via Proxmox UI: **Datacenter → Metric Server → Add → InfluxDB**

| Field | Value |
|-------|-------|
| Server | `10.0.60.41` (worker-monitoring-01) |
| Port | `8086` |
| Protocol | HTTP |
| API Version | 2 |
| Organisation | `home` |
| Bucket | `proxmox_metrics` |

Proxmox pushes on 1-minute interval. Data confirmed arriving in InfluxDB Data Explorer.

Dashboard imported: **ID 15356** (Proxmox Cluster [Flux] by mephisto) — showing live Proxmox host, VM, and ZFS data.

> [!note] Known limitation
> Proxmox does not report Container (LXC) disk I/O correctly — long-standing upstream bug. Don't debug missing disk I/O panels on LXC rows.

### Loki Data Source

- Connected: `http://monitoring_loki:3100`
- Dashboard imported: **ID 13639**

---

### Monitoring Stack — Final State

| Source | Pipeline | Status |
|--------|----------|--------|
| Traefik | → Prometheus → Grafana (ID 17346) | ✅ |
| Node Exporter (all nodes) | → Prometheus → Grafana (ID 1860) | ✅ |
| cAdvisor (all nodes) | → Prometheus → Grafana | ✅ |
| Proxmox host | → InfluxDB → Grafana (ID 15356) | ✅ |
| Container logs | → Loki → Grafana (ID 13639) | ✅ |

> [!todo] Remaining monitoring tasks (non-blocking, defer to Phase 8)
> - Prometheus targets audit — confirm node-exporter and cAdvisor UP for all 3 nodes
> - Pin `influxdb:2.7` in `stack-monitoring.yml` (currently `influxdb:latest`)
> - Dedicated Traefik metrics entrypoint `:8082`
> - Confirm monitoring overlay network joined to traefik stack (not just IP-based scraping)

---

## 🏗️ Media Architecture — Decisions Made

### VM Layout (confirmed)

| VM | VLAN | Second Disk | Label | Services |
|----|------|-------------|-------|---------|
| `worker-media-01` | 50 + 100 | 100G `/mnt/transcode` | `type=media` | Plex, Tautulli |
| `worker-mediamanagement-01` | 50 + 100 | 500G `/mnt/downloads` | `zone=mediamanagement` | Sonarr, Radarr, Prowlarr, Transmission+Gluetun |

Both VMs:
- Primary NIC: VLAN 50 (Media)
- Secondary NIC: VLAN 100 (Storage) — for direct NFS to Synology NAS at 10.0.100.20
- Services routed through Traefik; Plex also external via Cloudflare

**Transcode disk decision:** Local disk (not tmpfs) chosen — multiple simultaneous streams expected, local disk avoids RAM pressure and provides predictable performance.

### Data Flow

```
Transmission downloads → /mnt/downloads (local, worker-mediamanagement-01)
         │
         ▼  Sonarr/Radarr import on completion
Sonarr/Radarr copies → /mnt/media (NFS, Synology via VLAN 100)
         │  ← one copy, local disk → NFS
         ▼  Webhook to plex:32400 via media overlay
Plex reads /mnt/media (NFS read-only, worker-media-01)
         │
         ▼  Stream to client
```

### NFS Export Strategy

Both VMs on VLAN 50 (10.0.50.0/25) — export is read-write at the NFS level, read-only enforced at the mount level in the stack:
```
/mnt/media  10.0.50.0/25(rw,sync,no_subtree_check,no_root_squash)
```

---

## 💻 Code Session — Branch `claude/focused-feynman-ea3eba`

All committed and pushed. Files created:

| File | Purpose |
|------|---------|
| `scripts/02-provision-vm.sh` | Updated — added `--nic2-vlan`, `--disk2-size`, `--disk2-mount` flags |
| `scripts/03-post-boot.sh` | Updated — second disk init, NFS mount, `--nfs-mode ro\|rw`, `--skip-docker`, `--skip-swarm` flags |
| `scripts/migrate-media.sh` | ZFS send/receive — Plex + Tautulli configs to worker-media-01 |
| `scripts/migrate-mediamanagement.sh` | ZFS send/receive — Sonarr/Radarr/Prowlarr/Transmission to worker-mediamanagement-01 |
| `scripts/create-media-network.sh` | Creates `media` overlay network on manager-01 |
| `stacks/plex/stack-plex.yml` | Plex + Tautulli stack — `type=media` placement |
| `stacks/arr/stack-arr.yml` | Arr stack + Transmission+Gluetun — `zone=mediamanagement` placement |
| `docs/nfs-exports.md` | NFS export reference for Proxmox host |

**worker-media-01 provisioned:**
- VM created ✅
- SSH access ✅
- Docker installed ✅
- Joined Swarm ✅
- Primary disk (40G) ✅
- VLAN 50 (eth0) — IP assigned ✅

---

## 🚨 Blocking Issue — VLAN 100 / NFS

**Symptom:** `eth1` on `worker-media-01` has no IP address. NFS mount fails — traffic falls back to VLAN 50 source IP, denied by Synology export ACL (scoped to `10.0.100.0/24`).

**Diagnosis:**
- pfSense DHCP reservation set for MAC `52:54:e4:36:ea:e3` on VLAN 100 ✅
- Packet capture shows **no DHCP Discover reaching pfSense** on VLAN 100
- Root cause: VLAN 100 tag likely not being passed to the VM correctly

> [!bug] Bug — VLAN 100 not reaching worker-media-01
> DHCP Discover from eth1 not seen on pfSense VLAN 100 interface. Likely a Proxmox bridge or trunk port misconfiguration.

**Next session fix sequence:**

```bash
# Step 1 — Verify net1 tag on Proxmox host
qm config <vmid> | grep net1
# Expected: net1: virtio=52:54:e4:36:ea:e3,bridge=vmbr0,tag=100

# Step 2 — Verify VLAN 100 is in the allowed trunk list on the Extreme switch
# Check port facing Proxmox — VLAN 100 must be tagged

# Step 3 — Verify pfSense VLAN 100 interface is up and DHCP scope is active
# Interfaces → VLAN100_STORAGE → confirm enabled
# Services → DHCP Server → VLAN100 → confirm active, reservation present

# Step 4 — Once eth1 gets IP, re-run post-boot without re-doing Docker/Swarm
./proxmox-swarm/scripts/03-post-boot.sh worker-media-01 --nfs-mode ro --skip-docker --skip-swarm

# Step 5 — Verify NFS mount
mount | grep nfs
ls /mnt/media
```

---

## 📋 Next Session — In Order

1. **Fix VLAN 100 on worker-media-01** — debug trunk/bridge config, get eth1 IP, verify NFS mount
2. **Provision worker-mediamanagement-01** — same dual-NIC pattern, 500G downloads disk
3. **Merge branch** — `claude/focused-feynman-ea3eba` → main once both VMs are healthy
4. **Deploy stacks** — `create-media-network.sh` → `stack-plex.yml` → `stack-arr.yml`
5. **End-to-end test** — add torrent → Transmission → Sonarr import → Plex webhook → stream

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]]
- [[Plex Worker Storage & Network Layout]]
- [[VLAN and Subnet Summary Sheet]]
- [[pfSense Firewall Rules]]
- [[01 Homelab Rebuild - Phase 6 Service Deployment Hub]]
