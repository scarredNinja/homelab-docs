---
date: 2026-05-14
project_id: Homelab-2025
phase: "Phase 7: Backup Verification"
session_type: Bugfix + Implementation
status: Partial — restic NFS fix running overnight, Grafana scrape pending
tags:
  - Backup
  - Restic
  - NFS
  - Grafana
  - Prometheus
  - Monitoring
---

# Session Notes — 2026-05-14 — Restic Crash Diagnosis & NFS Benchmark

## Session Goal

Diagnose restic-backup.sh line 118 failure, fix root causes, and implement
NFS performance benchmarking with Grafana visibility.

---

## Completed

### restic-backup.sh — Stale Lock Auto-Heal (PR #33)

Script had no mechanism to clear stale restic locks left by failed runs.
Every nightly run after a crash hit `repository is already locked` and
aborted immediately. Fix: `restic unlock --remove-all` added at script
startup — no-op when clean, auto-heals when stale.

Additional fix in same PR: restic stdout+stderr now captured together into
the temp file so all output (snapshot IDs, Fatal messages) gets timestamps
in the log.

Stale lock from May 13 crash cleared manually this session using:
  restic -r rest:http://10.0.60.40:8000/ --password-file /etc/restic-password unlock --remove-all

Snapshot inventory confirmed clean — 5 snapshots present (May 7, 8, 10, 11, 13).
Gaps on May 9 and May 12 are confirmed missed backups due to the stale lock.

> [!bug] Bug — restic stale lock causes all subsequent nightly runs to abort (Gotcha #67)
> Once restic fails mid-run, the lock persists in the repo. Every subsequent
> backup attempt exits immediately with "repository is already locked". Script
> had no auto-heal. Fixed by calling `restic unlock --remove-all` at startup.
> See Runbook Appendix D Gotcha #67.

### restic-rest-server — NFS Mount Options Fixed (stack-restic.yml)

Container was crashing mid-backup due to `soft,timeo=30` NFS mount options
(3 second timeout). Under a 184 GiB write, any NFS hiccup >3s returned EIO
to the container, crashing it. Swarm rescheduled 3 times in quick succession
before stabilising. Backup client lost connection mid-transfer.

Confirmed via `docker service logs restic_restic-rest-server`:
  ERROR: sync /data/data/6f/....rest-server-temp...: input/output error

Fixed by changing NFS volume driver options in stack-restic.yml:
  Before: addr=10.0.100.20,nfsvers=3,rw,soft,timeo=30
  After:  addr=10.0.100.20,nfsvers=3,rw,hard,timeo=600,retrans=5

`hard` retries indefinitely on timeout instead of returning EIO.
timeo=600 = 60 seconds per retry attempt. retrans=5 = 5 retries.
Deployed — running overnight to verify.

> [!bug] Bug — restic-rest-server NFS mount `soft,timeo=30` crashes container under load (Gotcha #68)
> 3-second NFS timeout is too short for heavy write load. Any network hiccup
> during a large backup transfer causes EIO → container crash → Swarm reschedule
> → backup client disconnected. Changed to `hard,timeo=600,retrans=5`.
> See Runbook Appendix D Gotcha #68.

> [!warning] restic-rest-server running with --no-auth
> Authentication is disabled and port 8000 is published on the Swarm ingress
> (accessible on all node IPs). Any host that can reach a Swarm node on port
> 8000 can read or write backup data. Not resolved this session — added to
> backlog.

### nfs-bench.sh — NFS Performance Benchmark Script (PR #35)

New script deployed to worker-media-01. Tests:
  1. iperf3 to NAS IP — raw network ceiling
  2. dd sequential write — 128M blocks, conv=fdatasync
  3. dd sequential read — page cache dropped first
  4. fio random read — 4K bs, 30s, numjobs=4, iodepth=32
  5. fio sequential read — 1M bs, 30s

Output: colour-coded table to stdout, JSON append log at
/var/log/nfs-bench.log, Prometheus metrics to textfile collector.

Monthly cron: /etc/cron.d/nfs-bench on worker-media-01.
node-exporter textfile collector wired up — required full Portainer stack
restart to pick up the new textfile_collector path.

NFS Performance row merged directly into backup-status.json Grafana dashboard
(5 stat panels: seq read, seq write, network, random IOPS, random latency).

> [!warning] Grafana showing "No data" for NFS bench panels
> .prom file confirmed present on worker-media-01 and node-exporter is serving
> it. Likely cause: Prometheus not scraping 10.0.50.50:9100. Diagnostic
> command ready — see Next Session Priorities. See Gotcha #69.

---

## Current State

| Component | Status |
|-----------|--------|
| restic stale lock auto-heal | ✅ Fixed — PR #33 |
| restic-rest-server NFS mount | ✅ Fixed — deployed, overnight verification pending |
| Stale lock cleared manually | ✅ Done |
| nfs-bench.sh | ✅ Deployed to worker-media-01 |
| NFS bench monthly cron | ✅ Active |
| Grafana NFS Performance row | ✅ Panels exist — no data yet |
| Prometheus scraping worker-media-01:9100 | ❓ Unconfirmed — next session |
| restic-rest-server auth | 🔲 Backlog |

---

## Next Session Priorities

- [x] Verify overnight restic backup — check Grafana backup dashboard and Discord [priority::1]
- [x] Confirm restic-rest-server container stayed up (no reschedules in `docker service ps`) [priority::1]
- [x] Fix Prometheus scrape for worker-media-01 (10.0.50.50:9100): ✅ 2026-05-16

   docker exec $(docker ps -q --filter name=monitoring_prometheus) \
     wget -qO- 'http://localhost:9090/api/v1/targets' | python3 -c "
   import json,sys
   data = json.load(sys.stdin)
   for t in data['data']['activeTargets']:
       if 'node' in t['labels'].get('job',''):
           print(t['labels'].get('instance'), t['health'], t.get('lastError','')[:80])
   "

   If 10.0.50.50:9100 is missing → update prometheus scrape config and redeploy

 - [ ] restic-rest-server auth — add user, enable authentication [priority::2]

---

## Related Notes

- [[Docker Swarm Infrastructure Runbook]] — Appendix D Gotcha #67, #68, #69
- [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]
- [[New Backup Strategy]]
- [[Session Notes — 2026-05-13 — Backup Script Fixes & Disk Monitoring]]
