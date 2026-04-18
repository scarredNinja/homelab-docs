---
status: Planned
priority: Low
due_date: 2026-05-30
project_id: Homelab-2025
phase: "Phase 9: External Access"
---
# ⏸️ Phase 9: External Access

The final step for connectivity: setting up secure remote access and necessary port forwarding (VPN or Reverse Proxy).

## 🎯 Goals
## Verification & Testing
- [ ] Test IoT device connectivity with HA + Unifi after VLAN migration.  
- [ ] Test guest VLAN isolation (ensure VLAN 30 can only access allowed resources).  
- [ ] Test DMZ isolation (reverse proxy can’t directly access VLAN 60/65 without rules).  
- [ ] Confirm management VLAN 90 is reachable only from Admin VLAN 70 + VPN.  
- [ ] Run a connectivity matrix test (ping/traceroute across VLANs per rules).  

## Documentation
- [ ] Update Obsidian VLAN documentation with new firewall rules.  
- [ ] Document which services/containers live on which VLANs.  
- [ ] Link VLAN docs to service diagrams for quick lookup.


* Establish a secure VPN solution for remote access (e.g., WireGuard, OpenVPN).
* Configure dynamic DNS and necessary port forwarding on the edge firewall.
* **CRITICAL:** Ensure only necessary ports are exposed, and security policies are enforced.

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

- [ ] Deploy and configure the WireGuard VPN server.
- [ ] Set up dynamic DNS updates.
- [ ] Audit the firewall rules to block all unnecessary external access.

## 📝 Spoke Notes & Documentation
* [[WireGuard Configuration Log]]
* [[External Access Security Audit]]

---
## ➡️ Next Phase
* [[01 Homelab Rebuild - Phase 10 Documentation Hub]]