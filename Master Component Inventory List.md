---
project_id: Homelab-2025
phase: 'Phase 1: Preparation'
tags:
  - Hardware
  - Inventory
date: '2026-04-21T00:00:00.000Z'
status: Reference
---

# Master Component Inventory List

> Central hardware reference for the homelab rebuild. Update when hardware changes.

---

## Primary Server — HPE DL360p Gen8

| Spec | Detail |
|---|---|
| CPU | 2x Intel Xeon (confirm core count) |
| RAM | 128 GB |
| NICs | 4x 1 Gbps onboard |
| Form factor | 1U rack |
| OS | Proxmox VE |

### Storage Layout

| Logical Drive | RAID | Size | Bays | Media | Purpose |
|---|---|---|---|---|---|
| LD 01 | RAID 0 | 558 GiB | 1I Box 1 Bay 1 | 600 GB HDD | Data LUN |
| LD 02 | RAID 5 | 1117 GiB | 1I Box 1 Bays 2–4 | 1.2 TB + 600 GB HDDs | Data LUN |
| LD 03 | RAID 0 | 838 GiB | 2I Box 1 Bay 6 | 900 GB HDD | Data LUN |
| LD 04 | RAID 5 | 999 GiB | 2I Box 1 Bays 5–6 | 2× 1 TB SSD | Available |
| Free | — | — | 2I Box 1 Bays 7, 8 | — | Available |

**Raw drives:**
- 3× 600 GB SAS
- 2× 900 GB SAS
- 2× 1 TB SSD

> [!note] Confirm RAID on OS volume — believed to be RAID 5 but unverified.

---

## Managed Switch — Extreme Networks X440-48p

| Spec | Detail |
|---|---|
| Model | X440-48p |
| Ports | 48× 1 Gbps + uplinks |
| Firmware | EXOS 16.2 |
| PoE | Yes (48p = PoE on all ports) |
| Role | Core switch — all VLAN trunking |

---

## NAS — Synology

| Spec | Detail |
|---|---|
| Usable capacity | ~16.2 TB |
| IP | 10.0.100.20 (VLAN 100 — Storage) |
| Primary use | Media library NFS share (`/mnt/media`) |
| NFS export | `10.0.50.0/25` (Media VLAN) |

---

## VM Infrastructure (Proxmox)

| VM | Role | VLAN(s) | Swarm Role |
|---|---|---|---|
| `manager-01` | Swarm manager, Portainer, core services | 60 | Manager |
| `traefik-dmz-01` | Traefik reverse proxy | 60 + 80 | Worker (public zone) |
| `worker-monitoring-01` | Prometheus, Grafana, Loki, InfluxDB | 60 | Worker (monitoring zone) |
| `worker-media-01` | Plex, Tautulli | 50 + 100 | Worker (media zone) |
| `worker-mediamanagement-01` | Sonarr, Radarr, Prowlarr, Transmission | 50 + 100 | Worker (mediamanagement zone) |
| `worker-general-01` | General workloads | 40 | Worker (homelab zone) — planned |
| `worker-controller-01` | Home Assistant, UniFi | TBD | Worker (controller zone) — planned |

See [[Swarm Topology]] for current live status of each VM.

---

## ZFS Datasets (Proxmox Host)

| Dataset | Mount | Purpose |
|---|---|---|
| `rpool/docker-data` | `/mnt/docker-data` | Container config/data (virtiofs to VMs) |
| `rpool/docker-tsdb` | `/mnt/docker-tsdb` | Prometheus TSDB |
| `rpool/docker-swarm` | `/mnt/docker-swarm` | Swarm stack files |
| `rpool/docker-db` | `/mnt/docker-db` | Databases (Postgres, Redis) |

---

## Related

- [[Server Hardware]] — original hardware notes
- [[Swarm Topology]] — live VM status
- [[VLAN and Subnet Summary Sheet]] — network layout
- [[ZFS Configuration and Setup]] — ZFS detail
