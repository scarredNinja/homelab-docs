---
date: '2026-05-18T00:00:00.000Z'
project_id: Homelab-2025
phase: 'Phase 7: Backup Verification'
session_type: bugfix
status: Completed
tags:
  - backup
  - restic
  - nfs
  - cron
  - vm-backup
---

# Session Notes — 2026-05-18 — Restic NFS Hard Mount Fix

## Session Goal

Diagnose why restic-backup container was being SIGKILL'd mid-run after the
`hard` NFS mount option was applied in the previous session. Fix root cause
and prepare for a clean overnight run.

---

## Completed

### cron entry — vm-backup.sh was never scheduled

`vm-backup.sh` had no cron entry on the Proxmox host. The script existed and
was deployed but was never triggered by cron. Added the missing 03:00 entry
via `deploy-backup-scripts.sh`. This explains why VM backups had never run
in production. See Gotcha #48.

> [!bug] Gotcha #48 — vm-backup.sh cron entry missing since initial deploy
> Script was deployed but never added to crontab. Backup job never ran in
> production. Fixed by re-running `deploy-backup-scripts.sh` which writes
> all cron entries atomically. See Runbook Appendix D Gotcha #48.

### restic URL — incorrect IP corrected

restic-backup.sh was pointing to `10.0.60.40` (wrong host). Correct address
for restic-rest-server is `10.0.60.41`. Updated in script and confirmed.

### PR #38 (deferred retry) — confirmed live on `pve`

Sentinel-based deferral mechanism from the 2026-05-17 session confirmed
deployed on the Proxmox host. Branch `claude/restic-deferred-retry`.

### NFS `hard` mount — SIGKILL root cause diagnosed and fix pushed

The 2026-05-14 fix changed restic-rest-server NFS mount to `hard` to prevent
EIO errors under load. This introduced a new failure mode: under backup load,
any NFS stall causes the Go runtime to block all goroutines waiting on the
`hard` mount. The OOM killer or Swarm watchdog then SIGKILLs the container.
The `hard` option retries indefinitely with no timeout — there is no upper
bound on how long goroutines can be parked.

Fix: reverted to `soft` with conservative timeout values:
  `soft,timeo=50,retrans=3`

- `soft`: returns an error to the caller after retrans attempts instead of
  blocking forever
- `timeo=50`: 5-second timeout per attempt (timeo unit is 0.1s)
- `retrans=3`: 3 retry attempts before returning error

This gives restic-rest-server ~15 seconds to recover from a NFS hiccup
before returning an error, rather than blocking indefinitely. The Go runtime
can handle an error; it cannot handle a goroutine blocked forever.

Pushed to branch `claude/debug-backup-restic-hShHd` in `stack-restic.yml`.

> [!bug] Gotcha #49 — NFS `hard` mount blocks all Go goroutines under load
> Replacing `soft` with `hard` on the restic-rest-server NFS volume caused
> the Go runtime to park all goroutines indefinitely when NFS stalled. No
> timeout means no recovery path — container is SIGKILLed by the OS or Swarm.
> Fix: `soft,timeo=50,retrans=3`. This is a reversal of the Gotcha #68 fix
> and documents the correct balance between EIO risk and goroutine starvation.
> See Runbook Appendix D Gotcha #49.

---

## Tomorrow — Ordered Steps

1. Pull updated `stack-restic.yml` from branch `claude/debug-backup-restic-hShHd`
   on the Swarm manager and redeploy the restic stack via Portainer
2. Run `restic unlock` to clear any stale locks from tonight's expected failure
3. Run `restic-backup.sh` manually — confirm container survives the full run
4. If backup completes cleanly, merge PR #38 (deferred retry)

> [!note] Overnight cron tonight
> All four cron jobs will fire: 01:00 proxmox-config, 02:15 zfs send,
> 03:00 vm-backup, 05:00 restic. Restic will likely fail again tonight
> (stack not yet redeployed with NFS fix). Check logs before the manual
> test tomorrow — don't run `restic unlock` before reading them.

---

## Current State

| Component | Status |
|-----------|--------|
| vm-backup.sh cron | ✅ Fixed — entry now present |
| restic URL | ✅ Fixed — `10.0.60.41` |
| PR #38 deferred retry | ✅ Live on `pve` — pending merge after verification |
| NFS mount options fix | ⏳ Pushed — stack redeploy pending tomorrow |
| Overnight restic run | 🔲 Expected to fail — NFS fix not yet deployed |

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix D Gotcha #48, #49
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[Session Notes — 2026-05-17 — Restic Deferred Retry]]
- [[Session Notes — 2026-05-14 — Restic Crash Diagnosis & NFS Benchmark]]
- [[New Backup Strategy]]
