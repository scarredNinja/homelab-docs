---
project_type: Personal Fitness & Health App
project_id: FitnessDev-2026
status: Active
phase: Master Hub
tags:
  - project
  - fitness-app
  - homelab
---

# 🏋️‍♂️ Personal Fitness & Health - Master Project Hub

This note serves as the central directory for all physical health, training, nutrition, and biomarker tracking applications in your homelab. It dynamically tracks project phases, active tasks, and operational system inventory.

---

## 🚀 Phases Status (Tracking Progress)

This Dataview table automatically pulls the status, priority, and target dates from all individual project phase hubs.

```dataview
TABLE status, priority, due_date AS "Due Date"
FROM "10 - Projects"
WHERE contains(file.name, "Phase")
  AND contains(status, "Active") OR contains(status, "Planned")
  AND !contains(file.folder, "_Archive")
  AND project_id = "FitnessDev-2026"
SORT due_date ASC
```

---

## ✅ All Active Tasks (Across All Phases)

This list aggregates all incomplete tasks mapped to the `FitnessDev-2026` project ID, ordered by their inline priority tags.

```dataviewjs
// ── Fitness Dev active tasks — sorted by inline [priority:: N], unprioritised last ─
const PRIORITY_FALLBACK = 999;

const pages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id &&
        String(p.project_id).includes("FitnessDev-2026") &&
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

---

## 🌐 Active Service Access Mappings

*   **Local Development Server**: [http://localhost:8080](http://localhost:8080) *(Running locally in watch mode)*
*   **Production Swarm Service**: [https://newtonfit.home.purvishome.com](https://newtonfit.home.purvishome.com) *(Routed via Traefik secure entrypoint)*

---

## 🏗️ Architecture, Services & Strategy Docs

Use this section to navigate high-level architectural references and strategy runbooks.

*   **Active Dashboard Service**: [[Service - newtonfit]] — Live Docker Swarm deployment and production recovery guide.
*   **Target & Ingestion Roadmap**: [[NewtonFit - Data Ingestion & Target Sync Strategy]] — Technical roadmap for Syncthing, IMAP, and Tasker integrations.
*   **Host Environment**: [[VM - monitoring-01]] — Host VM pinning configurations on VLAN 60.

---

## 🗂️ Active Phase Hubs

*   **Phase 1**: [[05 Personal Fitness & Health - Phase 1 Ingestion & Target Sync Hub]] — Initial automation deployment for FitNotes, LoseIt!, and Samsung Health Sync.
