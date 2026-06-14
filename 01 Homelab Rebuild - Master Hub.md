---
project_type: Home Lab
project_id: Homelab-2025
tags:
  - HomeLabRebuild
status: Active
phase: Master Hub
---

# 💻 Homelab Rebuild - Master Project Hub

This note serves as the central directory for the Homelab Rebuild project, tracking the status and progress of all 10 phases. It automatically pulls data from all the individual Phase Hub notes (e.g., Phase 1, Phase 2, etc.).

---

## 🚀 Phases Status (Tracking Progress)

This Dataview table automatically pulls the status, priority, and due dates from all individual Phase Hubs. Update the metadata in the individual phase notes, and this table updates instantly.
```dataview
TABLE status, priority, due_date AS "Due Date"
FROM "10 - Projects"
WHERE project_id = "Homelab-2025" 
  AND contains(file.name, "Phase")
  AND contains(status, "Active") OR contains(status, "Planned")
  AND !contains(file.folder, "_Archive")
  AND project_id != "FinanceDev-2026" 
SORT due_date ASC
```
- [x] Review last change log [[Change Log#15/02/2026]]
- [x] Review last change log [[Change Log#24/02/2026]] ✅ 2026-03-02
## ✅ All Active Tasks (Across All Phases)

```dataviewjs
// ── Homelab active tasks — sorted by inline [priority:: N], unprioritised last ─
// 
// Your tasks use Dataview inline field syntax: [priority:: 1], [priority:: 2] etc.
// Lower number = higher priority (1 is most urgent).
//
// Sort logic:
//   1. Tasks with an explicit [priority:: N] field, ascending (1 first)
//   2. Tasks with no priority field at all, in file order
//
// The TASK query's built-in SORT can't do this split reliably, hence dataviewjs.

const PRIORITY_FALLBACK = 999; // pushes unprioritised tasks to the bottom

const pages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id &&
        String(p.project_id).includes("Homelab-2025") &&
        !p.file.folder.includes("_Archive") &&
        !p.file.folder.includes("Reference")
    );

// Collect incomplete tasks, filtering out #Later and #MuchLater tagged tasks.
// task.tags is an array of strings like ["#Hardware", "#Network"]
const tasks = pages
    .flatMap(p => p.file.tasks
        .where(t =>
            !t.completed &&
            !t.tags.includes("#Later") &&
            !t.tags.includes("#MuchLater") &&
            !t.tags.includes("#archive") 
        )
        .map(t => ({ task: t, file: p.file }))
    )
    .array();

// Sort: prioritised tasks first (ascending), unprioritised at the bottom
tasks.sort((a, b) => {
    const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
    const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
    if (pa !== pb) return pa - pb;
    // Stable secondary sort: file name then line number keeps the order predictable
    if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
    return (a.task.line ?? 0) - (b.task.line ?? 0);
});

// Render — dv.taskList renders proper interactive checkboxes
dv.taskList(tasks.map(t => t.task), false);
```

## 🏗️ Architecture & Configuration Notes (Global Links)

Use this section for essential, high-level documentation links that span across multiple phases.

- [[Network Architecture Overview]] (Your high-level design)
- [[Docker Swarm Infrastructure Runbook]]
- [[Swarm Topology]]
- [[High-Level Security Strategy]]
- [[Master Component Inventory List]]

## 🗂️ Swarm Inventory (Bases)

Live tables auto-populated from `Service - *.md` and `VM - *.md` frontmatter. Edit the underlying notes — these views update instantly.

### Swarm VMs

![[Swarm VM Board]]

### Swarm Services

![[Swarm Service Table]]

### Externally Accessible Services

![[Swarm External Services]]

## 📝 General Notes & Summaries

- Any notes about the project as a whole.
- Key lessons learned that might not fit into a single phase.
- [[10 - Projects/Passwords|Passwords]]
- [[Change Log]]
- [[Container Passwords]]
- [[Previous Container Backups]]

---
