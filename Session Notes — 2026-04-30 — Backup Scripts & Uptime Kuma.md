---
date: 2026-04-30
project_id: Homelab-2025
phase: "Phase 7: Backup Verification"
session_type: Implementation
status: Partial — pending Synology SSH fix and restic init
tags:
  - Backup
  - ZFS
  - Restic
  - UptimeKuma
  - DockerSwarm
---

# Session Notes — 2026-04-30 — Backup Scripts & Uptime Kuma

## Session Goal

Fix broken ZFS snapshot script, deploy Proxmox host config backup, scaffold Restic 
off-site backup to Synology, and fix + merge Uptime Kuma stack file.

---

## Completed

### Backup Scripts — Branch `claude/exciting-ride-82395a`

All scripts live in `proxmox-swarm/scripts/` in the 
[scarredNinja/docker-swarm-home](https://github.com/scarredNinja/docker-swarm-home) repo.

#### ZFS Snapshot Script Fixed

`zfs-snapshot.sh` was silently failing since 2026-04-10 due to wrong pool name.
See > [!bug] below and Gotcha #50 in Runbook Appendix D.

Fixed behaviour:
- Sends `rpool/docker-*` snapshots to `MainStorage/backups/<dataset>` daily at 02:00
- Incremental sends where prior snapshot exists, full send on first run
- Auto-creates destination datasets
- Discord webhook alert on failure
- Logs to `/var/log/zfs-snapshot.log`

Confirmed working: ZFS send running, all 4 datasets targeted.

#### Proxmox Host Config Backup — Created

`proxmox-config-backup.sh` — runs daily at 01:00.

Covers: `/etc/pve`, `/etc/network/interfaces`, `/etc/cron.d`, `/etc/fstab`, 
`/usr/local/bin`, and more. Archives to `MainStorage/backups/proxmox-host/` 
with 30-day retention. Discord webhook on failure.

Confirmed working: 25K archive created and logged clean.

#### Restic Backup Script — Created (pending Synology SSH fix)

`restic-backup.sh` — runs daily at 03:00.

Backs up `/mnt/docker-data`, `/mnt/docker-db`, `/mnt/docker-swarm` to Synology 
NAS over SFTP. Excludes Plex Cache/Codecs/Crash Reports. Retention: 7 daily, 
4 weekly, 3 monthly. Uses `Host synology-backup` SSH config alias.

NOT YET CONFIRMED — blocked on Synology sshd config issue (see Next Session).

#### Deploy Script — Created

`deploy-backup-scripts.sh` — one-shot deploy: installs all 3 scripts to 
`/usr/local/bin/`, writes `/etc/cron.d/backup-jobs`, runs preflight checks.
Accepts `--init-restic` flag to initialise restic repo on first run.

#### Cron Schedule

See `proxmox-swarm/cron/cron-jobs.md` in repo for exact cron entries.

| Time | Script |
|------|--------|
| 01:00 daily | `proxmox-config-backup.sh` |
| 02:00 daily | `zfs-snapshot.sh send daily` |
| 03:00 daily | `restic-backup.sh` |

---

### Uptime Kuma Stack — PR #16 Merged

File moved from `proxmox-swarm/stacks/monitoring/` to `proxmox-swarm/stacks/` 
to match repo convention. Four issues fixed vs sibling stacks:

- Removed double-quoted version string
- Added `failure_action: rollback` and `delay: 10s` to `update_config`
- Removed invalid `traefik.swarm.network` label
- Switched entrypoint from `internal` to `websecure`

Merged via PR #16. Not yet deployed — pre-deploy checklist pending.

---

## Bugs

> [!bug] Bug — ZFS snapshot script used non-existent pool name (Gotcha #50)
> `zfs-snapshot.sh` deployed 2026-04-10 with `DSTPOOL="oldpool"`. Correct pool 
> is `MainStorage`. Script ran nightly via cron but ZFS send silently failed — 
> piped commands don't propagate receive errors without `set -euo pipefail`. 
> No log file existed. Discovered 2026-04-30 via `zpool list`. No backups had 
> been taken since deployment. See Runbook Appendix D Gotcha #50.

> [!warning] Synology sshd_config has duplicate `Match User backup-agent` blocks
> Causes SSH auth failure for restic SFTP connection. Needs manual fix on 
> Synology before restic can run. See Next Session Priorities.

> [!warning] Discord webhook URL exposed in terminal output
> Webhook used for backup alerts was visible in session output. Rotate in Discord 
> before next session.

---

## Current State

| Component                  | Status                                 |
| -------------------------- | -------------------------------------- |
| ZFS snapshot → MainStorage | ✅ Running                              |
| Proxmox host config backup | ✅ Running                              |
| Restic → Synology          | ⏳ Blocked — Synology SSH fix needed    |
| Uptime Kuma stack          | ⏳ Merged, pre-deploy checklist pending |
| PBS                        | 🔲 Not started                         |

> [!note] MainStorage pool state
> `MainStorage`: 1.09T total, 390G used, 722G free.
> `MainStorage/backups` — active, receiving ZFS sends and host config archives.
> `MainStorage/downloads` — 390G, virtiofs-mapped to worker-mediamanagement-01.

---

## Next Session Priorities

- [x] 1. Confirm `zfs-snapshot.sh send daily` — verify all 4 datasets show `SEND OK` ✅ 2026-05-07
- [x] 2. Verify `prune hourly` and `prune daily` — confirm oldest snapshots removed ✅ 2026-05-07
- [x] 3. Run `proxmox-config-backup.sh` and verify archive output ✅ 2026-05-07
- [x] 4. Merge PR #15 once all three pass ✅ 2026-05-07 (superseded by PR #17)
- [x] 5. Fix Synology sshd_config duplicate `Match User` blocks, restart sshd ✅ 2026-05-07
- [x] 6. Run `deploy-backup-scripts.sh --init-restic` to initialise restic repo ✅ 2026-05-07
- [x] 7. Rotate Discord webhook URL ✅ 2026-05-07
- [x] 8. Grafana dashboards — import 11311, 11315, 11312 (unifi-poller) ✅ 2026-05-07
- [ ] 9. UniFi 404 — check Traefik logs
- [x] 10. Uptime Kuma — deployed via Portainer ✅ 2026-05-08
- [x] 11. Review overall backup strategy — PBS decided against; current chain sufficient ✅ 2026-05-07
- [x] 12. Add Portainer agent to `worker-mediamanagement-01` ✅ 2026-05-07 — Portainer agent unusable on Swarm workers (Gotcha #54); replaced with `tecnativa/docker-socket-proxy` on port 2375, registered as Docker API environment

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Steps 0.2, 0.2a, 0.2b, Appendix D Gotcha #50
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[New Backup Strategy]]
- [[Session Notes — 2026-04-26 — rpool Full Recovery & Downloads Migration]]
