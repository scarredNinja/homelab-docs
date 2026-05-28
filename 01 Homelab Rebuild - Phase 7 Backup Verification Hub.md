---
status: Planned
priority: Low
due_date: 2026-05-30
project_id: Homelab-2025
phase: "Phase 7: Backup Verification"
---
# 📅 Phase 7: Backup Verification

## 🎯 Goals

* Deploy a centralized monitoring solution (e.g., Prometheus/Grafana, Zabbix).
* Configure alerts for critical failures (e.g., disk full, node offline).
* Create a high-level dashboard for quick status checks.


Ensuring all critical data and configurations are backed up and, more importantly, can be successfully restored.

## 🎯 Goals

* Implement a reliable backup strategy (e.g., VeeaM, Duplicacy) for VMs and container data.
* **CRITICAL:** Perform a test restore of a critical service (e.g., Pi-Hole configuration).
* Document the full recovery process (Runbook).

## 🔗 Action Items

```dataviewjs
// ── Tasks for this phase only ─────────────────────────────────────────────────
// Uses dv.current().phase so this block is identical across every phase hub.
// It will only show tasks from files whose `phase` frontmatter matches this file's.
const PRIORITY_FALLBACK = 999;
const thisPhase = dv.current().phase;

if (!thisPhase) {
    dv.paragraph("⚠️ No `phase` field found in this file's frontmatter.");
} else {
    const pages = dv.pages('"10 - Projects"')
        .where(p =>
            p.phase === thisPhase &&
            !p.file.folder.includes("_Archive")
        );

    const tasks = pages
        .flatMap(p => p.file.tasks
            .where(t =>
                !t.completed &&
                !t.tags.includes("#Later") &&
                !t.tags.includes("#MuchLater")
            )
            .map(t => ({ task: t, file: p.file }))
        )
        .array();

    if (tasks.length === 0) {
        dv.paragraph("✅ No open tasks for this phase.");
    } else {
        tasks.sort((a, b) => {
            const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
            const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
            if (pa !== pb) return pa - pb;
            if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
            return (a.task.line ?? 0) - (b.task.line ?? 0);
        });

        dv.taskList(tasks.map(t => t.task), false);
    }
}
```

### Proxmox Backup Server

- [x] Check if needed — PBS decided against ✅ 2026-05-07 — ZFS send + vzdump + restic covers all layers. See Runbook Step 0.3. #Backup #PBS
- [x] **Configure PBS Sync** — not required; restic handles off-site via Synology NFS ✅ 2026-05-07 #Backup #Sync
- [x] Fix ZFS snapshot script pool name and verify sends (Gotcha #50) #Backup #Critical ✅ 2026-04-30

### Docker & Application Backups

- [x] Deploy ZFS snapshot → MainStorage backup script ✅ 2026-04-30 #Backup #ZFS
- [x] Deploy Proxmox host config backup script ✅ 2026-04-30 #Backup #Proxmox
- [x] Deploy VM backups (vzdump, all 6 VMs) via `vm-backup.sh` ✅ 2026-05-07 — PR #17 #Backup #VMs
- [x] Deploy Restic → Synology SFTP backup ✅ 2026-05-11 — switched from SFTP to restic-rest-server over NFS (nfsvers=3), deployed as Swarm stack. PR #17. #Backup #Restic
- [x] Deploy disk-space-check.sh — hourly ZFS + Synology metrics to node_exporter textfile collector ✅ 2026-05-13 #Backup #Monitoring
- [x] Add Grafana storage row — MainStorage free, downloads %, backups used, Synology free ✅ 2026-05-13 #Backup #Monitoring #Grafana
- [x] Store restic-password in password manager #Backup #Security
- [x] Rotate Discord webhook URL ✅ 2026-05-11 (sourced from /etc/backup-alerts.conf) #Backup #Security
- [x] **Set up Cron Jobs:** Configure automated daily backup schedules #Backup #Automation
- [ ] **Implement Backup Verification:** Create scripts to check archive integrity #Backup #Verification
- [ ] Configure monthly backup verification cron job #Backup #Verification

### External Drive System

- [ ]  **Set up External Drive System:** Acquire and format 2x USB 3.0 drives (4TB each) #Backup #External 
- [ ]  Create mount points and fstab entries for external drives #Backup #Mounting 
- [ ]  **Implement External Drive Backup Script:** Create sync script for critical backups #Backup #Scripts
- [ ]  Configure cron for external backups (Wednesday & Sunday) #Backup #Automation 
- [ ]  **Plan External Drive Rotation:** Establish weekly rotation schedule #Backup #Rotation 
- [ ]  Implement monthly deep backup script (optional) #Backup #Comprehensive 


- [x] Deploy Grafana/Prometheus container stack. ✅ 2026-04-30
- [x] Configure hardware exporters (Node Exporter) on all servers. ✅ 2026-04-30
- [x] Set up alerts for 80% CPU usage and disk capacity warnings. ✅ 2026-05-13

### Backup Monitoring & Strategy Review

- [x] Review whether PBS is still required ✅ 2026-05-11 (decision: not required) #Backup #PBS
- [x] Add backup job failure alerting — Discord webhooks added to all 4 scripts ✅ 2026-05-07 #Backup #Alerting
- [ ] Add Uptime Kuma monitors for backup job health #Backup #Monitoring

## 🗓️ Working Sessions

| Date | Session | Type | Status | Notes |
|------|---------|------|--------|-------|
| 2026-05-14 | Restic Crash Diagnosis & NFS Benchmark | Bugfix + Implementation | Partial — overnight verification pending | [[Session Notes — 2026-05-14 — Restic Crash Diagnosis & NFS Benchmark]] |
| 2026-05-17 | Restic Deferred Retry | Bugfix | Pending verification — overnight run 2026-05-18 | [[Session Notes — 2026-05-17 — Restic Deferred Retry]] |
| 2026-05-18 | Restic NFS Hard Mount Fix | Bugfix | Pending verification — stack redeploy tomorrow | [[Session Notes — 2026-05-18 — Restic NFS Hard Mount Fix]] |
| 2026-05-20 | NFS Benchmark Multi-Mount Extension | Implementation | PR #47 opened — NFS multi-mount + Grafana split by host/mount | [[Session Notes — 2026-05-20 — Alertmanager Discord Alerting]] |
| 2026-05-22 | Restic NFS Mount Fix & Backup Verification | Bugfix + Verification | PR #50 merged — changed NFS mount from `hard,timeo=600,retrans=5` to `soft,timeo=50,retrans=3`; stale lock cleared on Proxmox host; backup confirmed working | [[Session Notes — 2026-05-22 — Restic NFS Fix & Homepage Dashboard]] |
| 2026-05-26 | Restic REST Server Auth Hardening | Security + Verification | PR #61 — `--no-auth` replaced with `--htpasswd-file` backed by Docker secret `restic_htpasswd`; Proxmox host `RESTIC_REPO` updated to include credentials; backup connectivity verified (5 snapshots confirmed) | — |
| 2026-05-27 | VM Backup Registry Fix + Alert Cleanup | Bugfix + Refactor | All 9 registry files missing `first_manager` field — managers silently skipped by `build_vm_list`. Fixed: `first_manager: true` on manager-01, `false` on manager-02/03. Ran `deploy-backup-scripts.sh` to install PR #58 registry-driven `vm-backup.sh`. PR #62 — removed success Discord pings from all 3 backup scripts; failure-only alerting. Overnight verification pending. | — |

---

## 📝 Spoke Notes & Documentation
* [[Backup Schedules and Retention Policies]]
* [[New Backup Strategy]]
* [[Previous Container Backups]]
* [[cron-jobs]]
* [[Restore Test Log]]

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]