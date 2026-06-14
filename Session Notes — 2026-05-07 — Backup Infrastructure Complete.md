---
date: '2026-05-07T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 7: Backup Verification'
session_type: Code
status: Completed
tags:
  - Backup
  - ZFS
  - Restic
  - DockerSwarm
  - Grafana
---

# Session Notes — 2026-05-07 — Backup Infrastructure Complete

## 🎯 Session Goal

Verify and complete the backup chain from the 2026-04-30 session: confirm ZFS snapshot sends, verify host config backup, fix Synology SSH issue, initialise restic off-site backup, and merge outstanding PRs.

---

## ✅ Completed

### PR #17 — Merged

All work merged via PR #17. Branch `claude/exciting-ride-82395a` closed.

#### New: `vm-backup.sh`

Nightly vzdump of all 6 VMs. Workers backed up first, `manager-01` last. Stop mode. 3 copies retained. VM 203 transcode disk excluded via `backup=0`. Discord notification on completion.

#### New: `stacks/stack-restic.yml`

`restic-rest-server` deployed as a Swarm stack. Data written to Synology NAS over NFS (`10.0.100.20:/volume1/docker-backups`, nfsvers=3). Exposed on port 8000 via routing mesh.

#### Updated: `restic-backup.sh`

Switched from SFTP to rest-server HTTP endpoint. Sources from `/MainStorage/backups/*` (covers ZFS datasets, VM archives, and host config archives). Retains last 3 restic snapshots. Success notification added.

#### Updated: `zfs-snapshot.sh`

Success notification added to `send daily` run.

#### Updated: `proxmox-config-backup.sh`

Success notification added.

#### Updated: `deploy-backup-scripts.sh`

Adds `vm-backup.sh`, updated cron schedule, auto-elevation with sudo, storage and vzdump preflight checks.

---

### Nightly Backup Schedule (Live)

| Time  | Job                                      | Destination                |
| ----- | ---------------------------------------- | -------------------------- |
| 01:00 | Proxmox host config backup               | `MainStorage/backups/`     |
| 02:15 | ZFS send daily → MainStorage             | `MainStorage/backups/`     |
| 03:00 | VM backups (vzdump, all 6 VMs)           | `MainStorage/backups/`     |
| 05:00 | Restic off-site → Synology               | `10.0.100.20:/volume1/docker-backups` |

---

### PBS — Evaluated and Decided Against

Proxmox Backup Server was reviewed. Current chain (ZFS send + VM vzdump + restic off-site) provides sufficient coverage. PBS adds operational overhead without meaningful benefit given the existing layers. Backlog items closed.

---

## 🐛 Issues Resolved

> [!bug] Scripts missing execute bit
> All three backup scripts lacked `+x`. Fixed in git (`chmod` applied in repo). `deploy-backup-scripts.sh` now sets permissions on install.

> [!bug] NFS mount failing — nfsvers=4 rejected by Synology
> `stack-restic.yml` initially specified `nfsvers=4`. Synology DSM does not reliably support NFSv4 for this mount path. Switched to `nfsvers=3`. See Gotcha #51.

> [!bug] Docker volume stuck with stale NFS v4 options
> After the nfsvers fix, the existing Docker volume retained the old v4 mount options and could not be updated in place. Force-removed the volume and redeployed the stack. See Gotcha #52.

> [!bug] Proxmox host could not reach restic-rest-server on Swarm VLAN
> `restic-backup.sh` runs on the Proxmox host (10.0.90.50) and pushes to the rest-server on VLAN 60. pfSense was blocking traffic from 10.0.90.50 to 10.0.60.0/24:8000. pfSense rule added. See Gotcha #53.

> [!note] Synology IP placeholder corrected
> `restic-backup.sh` had a placeholder IP. Corrected to `10.0.100.20`.

---

## 📊 Current State (End of Session)

| Component                    | Status                  |
| ---------------------------- | ----------------------- |
| ZFS snapshot → MainStorage   | ✅ Running               |
| Proxmox host config backup   | ✅ Running               |
| VM backups (vzdump)          | ✅ Running               |
| restic-rest-server (Swarm)   | ✅ Deployed              |
| Restic → Synology            | ✅ Running               |
| PBS                          | ❌ Decided against      |
| Uptime Kuma                  | ⏳ Pre-deploy pending    |

| Item from 2026-04-30 list    | Status                  |
| ---------------------------- | ----------------------- |
| 1 — ZFS snapshot verify      | ✅ Done                  |
| 2 — Prune verify             | ✅ Done                  |
| 3 — Host config backup verify| ✅ Done                  |
| 4 — PR #15 merge             | ✅ Done (superseded by PR #17) |
| 5 — Synology SSH fix         | ✅ Done                  |
| 6 — restic init              | ✅ Done                  |
| 7 — Rotate Discord webhook   | ✅ Done                  |
| 8 — unifi-poller dashboards  | ✅ Done                  |

---

## ✅ Also Completed — Portainer Standalone Environment (worker-mediamanagement-01)

Portainer standalone environment added for `worker-mediamanagement-01` to provide UI visibility over `compose-vpn.yml` (Transmission + Gluetun), which runs outside Swarm.

`portainer/agent` approach failed — agent always forces cluster mode on Swarm nodes and crashes when it can't reach a manager (see Gotcha #54). Switched to `tecnativa/docker-socket-proxy` (already used in this setup for Traefik).

- docker-socket-proxy deployed on port 2375 with `restart: unless-stopped`
- Registered in Portainer as **Docker API** environment (not Agent), TLS disabled
- pfSense rule: source `10.0.60.30` (manager-01 only) → `10.0.50.51:2375` TCP
- Stack deployment via Portainer not possible on Swarm worker (Portainer attempts Swarm deploy, which fails)
- Environment provides: container list, logs, start/stop — sufficient for monitoring compose-vpn

`compose-vpn.yml` confirmed to have `restart: unless-stopped` on both services. Post-reboot manual restart step removed from runbook.

> [!bug] Portainer agent unusable on Swarm worker nodes — see Gotcha #54

---

## ✅ Also Completed — Grafana Backup Dashboard (PR #18)

Full metrics pipeline built and merged.

- `write_prom_metrics` function added to all 4 backup scripts — writes `.prom` files to `/var/lib/node_exporter/textfile_collector/` on success and failure, preserving last success timestamp across failure runs
- `deploy-backup-scripts.sh` updated — installs `prometheus-node-exporter`, creates textfile dir, enables `--collector.textfile.directory` flag
- Prometheus scrape job added for Proxmox host at `10.0.90.50:9100` (requires pfSense rule VLAN 60 → 10.0.90.50:9100 — add manually)
- `backup-status.json` dashboard created at `proxmox-swarm/stacks/monitoring/dashboards/` — two rows: Last Run Status (green/red stat panels) and Last Success (humanized duration with 26h/50h thresholds)
- `copy-traefik-config.sh` renamed to `copy-swarm-config.sh` with expanded scope

> [!note] Partial verification — `proxmox_config_backup` confirmed in Prometheus. Remaining 3 jobs (`zfs_snapshot`, `vm_backup`, `restic_backup`) pending overnight cron run before metrics appear.

---

## ➡️ Next Session Priorities

- [x] Verify all 4 backup jobs visible in Prometheus and Grafana backup dashboard after overnight run [priority:: 1]
- [x] Add pfSense rule: VLAN 60 → `10.0.90.50:9100` TCP (Prometheus → Proxmox host node_exporter) [priority:: 1]
- [x] UniFi 404 — check Traefik logs to diagnose routing failure [priority:: 2]
- [x] Uptime Kuma — pre-deploy checklist and deploy via Portainer [priority:: 1]
- [x] Verify UniFi Prometheus metrics visible in Grafana [priority:: 2]
- [ ] Allow Home VLAN (VLAN 10 — `10.0.10.0/25`) access to Home Assistant: pfSense rule + Traefik `internal-only` middleware allowlist [priority:: 2]

---

## 🔗 Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Step 0.2, Step 0.3, Appendix D Gotchas #51–53
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[New Backup Strategy]]
- [[Session Notes — 2026-04-30 — Backup Scripts & Uptime Kuma]]
