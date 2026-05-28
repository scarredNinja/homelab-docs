---
status: "Active"
priority: "Low"
due_date: "2026-04-30" 
project_id: "Homelab-2025"
phase: "Phase 2: Physical Setup"
---
# ✅ Phase 2: Physical Setup

Focus on the physical installation of hardware, cabling, and basic power-on testing.

## 🎯 Goals

* Install and secure all components in the rack.
* Complete all network cabling and patch panel connections.
* Verify that all components (servers, switches, UPS) power on without error.

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
### Hardware Setup

- [x]  **Physical Server Setup:** Connect HPE Proliant DL360p Gen8 to network switch ✅ 2025-08-04 #Hardware
- [x] Connect iLO port and assign static IP on VLAN 90 (Management) [priority:: 1] #Hardware #Network ✅ 2026-01-28
- [x] add old network to new switch config to avoid change over time issue [priority:: 1] #Hardware #Network ✅ 2026-01-28
- [x] Create new cable from router to patch port one   [priority:: 1] #Hardware
- [ ] Look at second cable from router to waredrobe. What could I repurpose it for.   [priority:: 3] #Hardeare
- [x] Label all cables and patch panel ports  [priority:: 2]#Hardware
- [x] Repatch cables [priority:: 2]#Hardware
- [x] bring through cable to the lounge ✅ 2026-05-09
- [x] setup switch in lounge #hardware [priority:: 2]
- [ ] organize power cables #hardware [priority:: 2]
- [ ] create cable for pdu #Network #Hardware [priority:: 3] 
- [ ] Create cable for Pi 1 and patch [priority:: 3] #Hardware
- [ ] Create cable for Pi 2 and patch [priority:: 3] #Hardware
- [ ] Bring through cables for mine and kris' PC [priority:: 2] #Hardware 
- [x] Ask Doug about 3d printing rpi rack [priority:: 3] #Hardware
- [ ] look at Raspberry pis to add
- [ ] tidy up existing pis [[Physical Items To Look#Existing Pis]]
### Proxmox Installation

- [x]  **Proxmox VE Fresh Install:** Boot from ISO and install on RAID 10 storage ✅ 2025-08-03 #Proxmox #Installation
- [x]  Configure primary management interface with static IP on VLAN 90 ✅ 2025-08-03 #Proxmox #Network

- [x] Update Spoke Notes [priority:: 2]



## 📝 Spoke Notes & Documentation
* [[New Server Setup]]
* [[Rack Diagram v2]] (Reference the latest physical layout) - add when Netbox is setup
* [[Cable Management Log]]
* [[Physical Items To Look]]

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 3 Network Config Hub]]