---
status: Planned
priority: Low
due_date: 2026-05-30
project_id: Homelab-2025
phase: "Phase 7: Basic Monitoring"
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

- [ ]  **Configure PBS:** Set up Proxmox VE nodes to send backups to PBS on VLAN 60 #Backup #PBS 
- [ ]  **Configure PBS Sync:** Set up PBS to sync backups to Synology NAS #Backup #Sync

### Docker & Application Backups

- [ ]  **Implement Docker Volume/Config Backups:** Deploy backup tools (Duplicati/Restic) #Backup #Docker
- [ ]  **Set up Cron Jobs:** Configure automated daily backup schedules #Backup #Automation 
- [ ]  **Implement Backup Verification:** Create scripts to check archive integrity #Backup #Verification
- [ ]  Configure monthly backup verification cron job #Backup #Verification 

### External Drive System

- [ ]  **Set up External Drive System:** Acquire and format 2x USB 3.0 drives (4TB each) #Backup #External 
- [ ]  Create mount points and fstab entries for external drives #Backup #Mounting 
- [ ]  **Implement External Drive Backup Script:** Create sync script for critical backups #Backup #Scripts
- [ ]  Configure cron for external backups (Wednesday & Sunday) #Backup #Automation 
- [ ]  **Plan External Drive Rotation:** Establish weekly rotation schedule #Backup #Rotation 
- [ ]  Implement monthly deep backup script (optional) #Backup #Comprehensive 


- [ ] Deploy Grafana/Prometheus container stack.
- [ ] Configure hardware exporters (Node Exporter) on all servers.
- [ ] Set up alerts for 80% CPU usage and disk capacity warnings.

## 📝 Spoke Notes & Documentation
* [[Backup Schedules and Retention Policies]]
* [[New Backup Strategy]]
* [[Previous Container Backups]]
* [[Restore Test Log]]

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 8 Basic Monitoring Hub]]