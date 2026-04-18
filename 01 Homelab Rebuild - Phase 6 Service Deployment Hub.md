---
status: Planned
priority: Medium
due_date: 2026-05-01
project_id: Homelab-2025
phase: "Phase 6: Service Deployment"
---
# 📅 Phase 6: Service Deployment

Deploying the core applications and services that bring the Homelab to life (e.g., Media servers, Wiki, internal dashboards).

## 🎯 Goals

* Deploy all planned core services (e.g., Plex, Portainer, internal dashboard).
* Configure internal DNS and reverse proxy access for all services.
* Verify all persistent volumes are mapped correctly to the configured storage.

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

### Infrastructure Services

- [ ] **Deploy Proxmox Backup Server (PBS):** Create on VLAN 60, install and configure PBS [priority:: 2] #Backup #Infrastructure #Update

### Application Services

- [x] Update list of work services and arwas/VLANS #Update ⏫ ➕ 2025-09-25 ⏳ 2025-09-26 [priority:: 2] ✅ 2026-02-05
- [ ] Get a copy of all service stackds and save to GIT [priority:: 3]
- [ ]  **Deploy Media Services (VLAN 50):** Plex, qBittorrent, Tdarr with NFS mounts #Docker #Media 
- [ ]  **Deploy Media Management (VLAN 50):** Sonarr, Radarr, Prowlarr with NFS persistence #Docker #MediaMgmt 
- [ ]  **Deploy Homelab Services (VLAN 40):** Home Assistant, Grafana with NFS persistence #Docker #Homelab 
- [ ]  **Deploy Database Services (VLAN 65):** PostgreSQL, MariaDB, Redis with NFS persistence #Docker #Database 
- [ ]  **Deploy DMZ Services (VLAN 80):** External Access Dashboard (Dashy) with NFS persistence #Docker #DMZ 


- [x] Create Docker Compose stacks for all core services. ✅ 2026-02-15
- [x] Set up Traefik/Nginx Proxy Manager for internal routing. ✅ 2026-02-05
- [x] Test application access via internal hostnames (e.g., `plex.local`). ✅ 2026-02-05

## 📝 Spoke Notes & Documentation
- [[Docker Swarm Worker Storage Layout]]
* [[Traefik]]
* 

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 7 Backup Verification Hub]]