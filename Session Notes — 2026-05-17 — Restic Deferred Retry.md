---
date: '2026-05-17T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 7: Backup Verification'
session_type: bugfix
status: Completed
tags:
  - backup
  - restic
  - cron
  - vm-backup
  - sentinel
---

# Session Notes — 2026-05-17 — Restic Deferred Retry

## Session Goal

Fix restic-backup.sh silently skipping when vm-backup.sh overruns past the
05:00 restic cron window. Observed 2026-05-17: VMs finished at 05:16, restic
never re-triggered. No retry mechanism existed.

---

## Completed

### Sentinel-based deferral and retry mechanism (PR #38)

Three scripts modified, one doc updated. Branch: `claude/restic-deferred-retry`.

**`restic-backup.sh`**
Writes `/var/run/restic-deferred` sentinel when it detects the vm-backup lock
is held and defers. Removes the sentinel on successful completion. Added
missing `PATH` export (same class of bug as Gotcha #61 — cron PATH).

**`vm-backup.sh`**
At end of run: explicitly releases `/tmp/vm-backup.lock` and disarms the
EXIT trap, then checks for the sentinel. If found, calls `restic-backup.sh`
directly. Lock must be released before the restic call — otherwise restic
detects the lock, defers again, and writes a new sentinel, creating an
infinite loop. See Gotcha #71.

**`deploy-backup-scripts.sh`**
Adds an 08:00 safety-net cron entry. Fires only if the sentinel still exists —
guards against `vm-backup.sh` crashing before it reaches the trigger step.
Schedule now: 03:00 vm-backup, 05:00 restic, 08:00 safety-net.

**`cron/cron-jobs.md`**
Synced to reflect the actual three-job schedule and documents the deferral
pattern for future reference.

Deployed via `deploy-backup-scripts.sh` on the Proxmox host.

> [!warning] Overnight verification pending
> PR #38 not merged. Verify tomorrow morning that the 03:00 → 05:16 → restic
> chain fires correctly, or that the 08:00 safety-net triggers if needed.

> [!note] Gotcha #71
> Lock release ordering is critical — restic must not be called while
> vm-backup.lock is still held. See Runbook Appendix D Gotcha #71.

---

## Current State

| Component | Status |
|-----------|--------|
| restic deferral sentinel | ✅ Implemented — PR #38 |
| vm-backup.sh → restic trigger | ✅ Implemented — PR #38 |
| 08:00 safety-net cron | ✅ Deployed |
| Overnight verification | 🔲 Pending — verify 2026-05-18 |

---

## Next Session Priorities

1. Check logs and Grafana backup dashboard morning of 2026-05-18 — confirm full chain ran
2. Merge PR #38 after verification
3. Confirm restic snapshot count increased (proves full backup ran)

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix D Gotcha #71
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[New Backup Strategy]]
- [[Session Notes — 2026-05-14 — Restic Crash Diagnosis & NFS Benchmark]]
