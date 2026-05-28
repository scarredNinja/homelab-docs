---
date: 2026-05-13
project_id: Homelab-2025
phase: "Phase 7: Backup Verification"
session_type: Bugfix + Implementation
status: Complete — first fully automated cron run pending overnight
tags:
  - Backup
  - Restic
  - ZFS
  - Monitoring
  - Grafana
  - Discord
  - Proxmox
---

# Session Notes — 2026-05-13 — Backup Script Fixes & Disk Monitoring

## Session Goal

Diagnose why restic and VM backups were not running despite being deployed,
fix root causes, add disk-space monitoring with alerting, and consolidate
Discord notifications across all backup scripts.

---

## Completed

### restic-backup.sh — Preflight Check Fixed (Gotcha #65)

Script was using `curl --fail` to probe the restic REST server before running
backups. The REST server returns HTTP 405 on a plain `GET /` — this is expected
behaviour, not an error. `--fail` treats any non-2xx response as a failure and
exits non-zero, causing the entire backup to abort before a single file was read.

Fixed by removing `--fail` from the preflight curl. Restic ran successfully:
184 GiB backed up, forget/prune completed. Stale lock from a failed May 8 run
was also cleaned up.

> [!bug] Bug — restic preflight `--fail` exits on HTTP 405 (Gotcha #65)
> Every nightly restic backup had been aborting silently since first deployment.
> The REST server's 405 on `GET /` is correct — it only accepts POST. Removing
> `--fail` from the preflight curl fixed the issue. See Runbook Appendix D Gotcha #65.

### vm-backup.sh — PATH Fixed (Gotcha #66)

`qm` lives in `/usr/sbin`, which is not in cron's default PATH. The script
logged `START` then silently exited every night — `qm` was not found, the
command silently failed, and nothing in the error path triggered a Discord alert.

Fixed by adding `export PATH=/usr/local/sbin:/usr/sbin:/usr/bin:/sbin:/bin`
at the top of the script. All 6 VMs backed up successfully and confirmed via
Discord summary notification.

Stale restic lock from May 8 cleaned up during this session.

> [!bug] Bug — vm-backup.sh cron missing `/usr/sbin` in PATH (Gotcha #66)
> `qm` binary not found in cron environment. Script appeared to start (logged
> START) then silently did nothing. All 6 VM backups had been skipped since
> deployment. See Runbook Appendix D Gotcha #66.

### disk-space-check.sh — New Hourly Script

New script writing ZFS and Synology free-space metrics to node_exporter textfile
collector at `/var/lib/node_exporter/textfile_collector/`.

Metrics written per run:
- `MainStorage` pool free space (GiB)
- `downloads` dataset used and free (GiB + %)
- `backups` dataset used (GiB)
- Synology `/volume1` free space (GiB) — fetched via transient read-only NFS mount

Discord alerts fire at most **once per 24 hours** per threshold (flag file at
`/tmp/disk-alert-sent-<metric>`) when:
- downloads dataset >85% of quota
- MainStorage pool <50 GiB free
- Synology /volume1 <1 TiB free

Schedule: hourly via cron.

Infrastructure changes made to support Synology NFS fetch:
- pfSense rule added: VLAN 90 → Synology `10.0.100.20` NFS port 2049
- Synology NFS export updated: added `10.0.90.0/25` as allowed client

### Grafana — Storage Dashboard Row Added

New "Storage" row added to existing backup dashboard:
- MainStorage pool free (GiB)
- downloads free (GiB) + downloads used %
- backups used (GiB)
- Synology /volume1 free (GiB)

All panels sourced from textfile collector metrics via Prometheus.

### Notification Consolidation — All Scripts

`vm-backup.sh` and `zfs-snapshot.sh` previously sent one Discord alert per
failed item. All four backup scripts (`vm-backup.sh`, `zfs-snapshot.sh`,
`restic-backup.sh`, `proxmox-config-backup.sh`) now send exactly **one
notification per run** with a pass/fail summary. Reduces noise, ensures a
single alert is easy to triage.

---

## Verified Working

| Component | Status | Detail |
|-----------|--------|--------|
| Restic → Synology | ✅ Working | 184 GiB backed up, forget/prune complete |
| VM backup (vzdump) | ✅ Working | 6/6 VMs confirmed via Discord |
| disk-space-check.sh | ✅ Running | All ZFS + Synology values live in textfile collector |
| Grafana storage row | ✅ Live | All 4 panels populated |
| Notification consolidation | ✅ Applied | All scripts emit one summary per run |

---

## Current State

| Component | Status |
|-----------|--------|
| ZFS snapshot → MainStorage | ✅ Running |
| Proxmox host config backup | ✅ Running |
| Restic → Synology | ✅ Running — 184 GiB backed up |
| VM backup (vzdump) | ✅ Running — 6/6 VMs |
| disk-space-check.sh | ✅ Running hourly |
| Grafana storage row | ✅ Deployed |
| Cold storage (restic copy → Windows) | 🔲 Design agreed, not implemented |
| PBS | 🔲 Decided against — ZFS + restic covers all layers |

> [!note] Tonight is the first fully automated end-to-end cron run
> All three nightly backup jobs (ZFS, restic, vzdump) plus hourly disk-space-check
> will run unattended for the first time. Verify Grafana and Discord alerts
> tomorrow morning.

---

## Still To Do

- Cold storage: `restic copy` to Windows PC via SMB mount — design agreed, not yet implemented
- Verify overnight cron runs in Grafana and Discord
- `deploy-backup-scripts.sh` + `copy-swarm-config.sh` on Proxmox — already done this session

---

## Next Session Priorities

1. Verify overnight backup cron runs (Grafana dashboard + Discord notifications)
2. Cold storage implementation: `restic copy` to Windows PC via SMB mount
3. Diagnose Tailscale routing (`ping 10.0.60.1`, pfSense logs, `net.inet.ip.forwarding`)
4. pfSense rule VLAN60→`10.0.90.50:9100` (Prometheus→Proxmox node_exporter)
5. Deploy Uptime Kuma via Portainer (pre-deploy checklist pending)
6. Fix UniFi 404 via Traefik logs
7. VLAN 10→Home Assistant access

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix D Gotcha #65, #66
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[New Backup Strategy]]
- [[Session Notes — 2026-04-30 — Backup Scripts & Uptime Kuma]]
