---
status: Completed
priority: High
due_date: 2026-02-01
project_id: Homelab-2025
phase: "Phase 1: Preparation"
---
# ✅ Phase 1: Preparation

This phase focuses on the initial planning, component ordering, and final design decisions needed before any physical work begins.

## 🎯 Goals

* Finalize all component purchasing.
* Document the logical network design (VLANs, Subnets).
* Create a clean, centralized list of all necessary configuration files/notes.

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

### Backups & Documentation

- [x] **Backup Everything:** Perform full backups of any existing data and configurations. Verify the integrity of these backups. Copy critical data from your Synology NAS. #Backup #Critical ✅
- [x]  Create scripts to back up LXC container 104 data to NAS ✅ #Backup #Scripts
- [x]  Create scripts to back up all LXC container data to NAS ✅ #Backup #Scripts
- [x]  Test backup scripts with one container ✅  #Backup #Testing
- [x] Set daily backup schedule (replace PBS backups) ✅ #Backup #Automation
- [x] Confirm backups are running ✅ #Backup #Verification
- [x] Export all Portainer stacks ✅ #Backup #Portainer
- [x]  **Document Current Configuration:** Note down existing IP addresses, network settings, and software configurations. ✅ #Documentation 

### Planning & Design

- [x] **Network Diagram:** Draw detailed network diagram with all VLANs (10,20,30,40,50,60,70,75,80,90) and IP subnets #Network #Documentation 🔼 ✅ 2025-09-25
- [x] **IP Addressing Scheme:** Define static IPs for infrastructure and DHCP ranges #Network #Documentation  ⏫ ✅ 2025-08-22
- [x]  All pfSense VLAN Rules outlined and documented ✅ 2025-08-04 #Network #pfSense #Documentation #todoist
- [x] Update current networking config for pfSense and switch #Network #Documentation
- [x]  Update IP address scheme as configured ✅ 2025-08-04 #Network #Documentation 
- [/] Document pfSense rules #pfSense #Documentation ⏫ 🛫 2025-07-01 ⏳ 2025-08-15 📅 2026-06-19

### Infrastructure Preparation

- [x]  **Download ISOs:** Get latest Proxmox VE ISO ✅ 2025-08-04 #Proxmox
- [x]  Setup Bootable Proxmox USB ✅ 2025-08-04 #Proxmox
- [x]  Look at current storage setup and layout new server storage ✅ 2025-08-04 #Storage #Planning
- [x]  Get current RAID setup on rack server and expansion slots ✅ 2025-08-04 #Storage #Hardware
- [x]  Update notes with new hard drives and clean up ✅ 2025-08-04 #Storage #Documentation 
- [x]  Steps to setup RAID on server refresh ✅ 2025-08-04 #Storage #Documentation 
- [x] Get size of external hard drives  [priority:: 3] #Storage #Backup ✅ 2026-02-05

### Docker & Application Prep

- [x]  **Docker Installation Script/Notes:** Prepare commands for Docker Engine installation and Docker Swarm initialization ⏫ ✅ 2025-08-04 #Docker #Scripts
- [x]  Installation Steps for Docker Swarm (VMs vs LXCs decision) ✅ 2025-08-04 #Docker #Documentation
- [x]  Information on storage and persistence with Docker Swarm setup ✅ 2025-08-04 #Docker #Storage #Documentation

## 📝 Spoke Notes & Documentation
* [[Server Hardware]]
* [[Storage Analysis - Current]]
* [[Storage Migration]]
* [[Server Component List]]
* [[Initial Network Design Draft]]
* [[Homelab Rebuild Budget]] 

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 2 Physical Setup Hub]]