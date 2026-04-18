---
status: Completed
priority: "High"
due_date: "2026-02-01" 
project_id: "Homelab-2025"
phase: "Phase 4: Storage Config"
---
[[01 Homelab Rebuild - Master Hub]]

# 🔄 Phase 4: Storage Config

This phase focuses on configuring and optimizing the main storage solution for virtualization and containers (e.g., ZFS, RAID, Ceph).

## 🎯 Goals

* Successfully configure the main storage array (RAID/ZFS/etc.).
* Optimize storage performance for VMs (e.g., set appropriate block sizes).
* Create initial volumes/datasets for the container platform.

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

### Proxmox Network Setup

- [x] **Proxmox VE Network Configuration:** Access web interface and configure networking #Proxmox #Network #Critical  🔺 📅 2025-08-12 ✅ 2025-09-24
    - [x] **Create Linux Bond:** bond0 from 3 Broadcom NICs, Mode: 802.3ad (LACP) #Proxmox #Network #Bonding ✅ 2025-08-16 
    - [x] **Create Linux Bridge:** vmbr0, Bridge ports: bond0, enable "VLAN aware" #Proxmox #Network #Bridge ✅ 2025-08-16 
    - [x] **Create VLAN Bridges:** Confirm if i need too! For each VLAN (10,20,30,40,50,60,65,70,75,80,85,90) create vmbr0.X bridges #Proxmox #Network #VLAN  🔺 ⏳ 2025-08-05 ✅ 2025-08-24
    - [x] Update configuration based on current host connection issue [priority:: 1]
- [x] Update tables with FQDN and if SSL is working #Network [priority:: 1] ✅ 2026-02-05
- [x] Recreate the bond setup [[Proxmox Network Setup#Update - v2]] #Network [priority:: 1]

### Storage Configuration

- [x]  **Synology NAS Configuration:** Ensure NAS has static IP on VLAN 60 ✅ 2025-08-04 #Storage #NAS 
- [x] Add Proxmox storage configuration - [[Storage Migration]] [priority:: 1]#Proxmox #Storage #Documentation #Update ✅ 2026-01-16
- [x] Draw out storage setup diagram (check if I have a current one) [priority:: 2] #Storage #Documentation #Uodate ✅ 2026-01-16
- [x] **Configure NFS Permissions:** Set folder access for plex-01 worker vm #Storage #NFS #Permissions  ✅ 2025-08-24
- [x] Add SMB access to Proxmox for PC access #Storage #SMB #Access  ✅ 2025-08-24
- [x] Configure ZFS pool on the main storage server. ✅ 2025-12-15
- [x] Look at if I should start again with the proxmox install or just update as ✅ 2026-01-16
- [x] Check current storage use on new server ssd  [priority:: 1][[Storage Analysis - New]] ✅ 2026-02-03
- [x] Migrate to new storage/data setup [[Proxmox storage migration setup#Create specific children for your apps]]  [priority:: 1] ✅ 2026-02-03
- [x] Investigate Proxmox to Manager VM [[Proxmox Network Setup#Update]] [priority:: 1] ✅ 2026-02-14
- [x] Run basic read/write performance tests on the new pool. ✅ 2026-03-02
- [x] Document all mount points and share paths in the [[Storage Map]] note. [priority:: 2] #Storage ✅ 2026-03-02


```dataview 
TASK 
FROM "10 - Projects" 
WHERE !completed 
AND contains(phase, "Phase 4: Storage Config") 
AND file.name != this.file.name 
SORT file.name ASC 
```
## 📝 Spoke Notes & Documentation
* [[Plex Worker Storage & Network Layout]]
* [[NFS Mounting]]
* [[Storage Migration]]
* [[Proxmox Storage Assignment – Docker Swarm VMs]]
* [[Docker Swarm Worker Storage Layout]]
* [[ZFS Configuration and Setup]]
* [[Mount Point Map]]
* [[Proxmox storage migration setup]]
* [[Virtio-FS (ZFS Datasets from Proxmox Host)]]
---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 5 Docker Swarm & Virtualisation Hub]]