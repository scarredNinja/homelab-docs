---
project_id: FinanceDev-2026
status: Active
phase: Master Hub
tags:
  - project
  - finance-app
  - homelab
---

# 💰 Finance Dev - Master Project Hub

This note serves as the central directory for the Finance Dev project, tracking the status and progress of all phases. It automatically pulls data from all the individual Phase Hub notes.
## 🚀 Phases Status (Tracking Progress)

This Dataview table automatically pulls the status, priority, and due dates from all individual Phase Hubs. Update the metadata in the individual phase notes, and this table updates instantly.

```dataview
TABLE status, priority, due_date AS "Due Date"
FROM "10 - Projects"
WHERE contains(file.name, "Phase")
  AND contains(status, "Active") OR contains(status, "Planned")
  AND !contains(file.folder, "_Archive")
  AND project_id = "FinanceDev-2026"
SORT due_date ASC
```

## ✅ All Active Tasks (Across All Phases)

```dataviewjs
// ── Finance Dev active tasks — sorted by inline [priority:: N], unprioritised last ─
//
// Your tasks use Dataview inline field syntax: [priority:: 1], [priority:: 2] etc.
// Lower number = higher priority (1 is most urgent).
//
// Sort logic:
//   1. Tasks with an explicit [priority:: N] field, ascending (1 first)
//   2. Tasks with no priority field at all, in file order

const PRIORITY_FALLBACK = 999;

const pages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id &&
        String(p.project_id).includes("FinanceDev-2026") &&
        !p.file.folder.includes("_Archive")
    );

// Collect incomplete tasks, filtering out #Later and #MuchLater tagged tasks.
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

// Sort: prioritised tasks first (ascending), unprioritised at the bottom
tasks.sort((a, b) => {
    const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
    const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
    if (pa !== pb) return pa - pb;
    if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
    return (a.task.line ?? 0) - (b.task.line ?? 0);
});

dv.taskList(tasks.map(t => t.task), false);
```

## 🏗️ Architecture & Configuration Notes (Global Links)

Use this section for essential, high-level documentation links that span across multiple phases.

- [[Finance App - Architecture Overview]]
- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Finance App - Docker Compose Setup]]

## 📝 General Notes & Summaries

- Any notes about the project as a whole.
- Key lessons learned that might not fit into a single phase.
- [[Finance Dev Change Log]]
- [[Finance App - Environment Variables]]
- [[Finance App - Deployment Notes]]

---
