---
status: Reference
priority: Low
due_date: null
project_id: FinanceDev-2026
phase: 'Phase 3: TBD'
tags:
  - finance-app
---

# 🔮 Phase 3: TBD

Phase 3 is not yet scoped. This hub is a placeholder so the wikilink from Phase 2 resolves and the Dataview queries in the Master Hub and Change Log pick up this file correctly once tasks are added.

## 🎯 Goals

- To be defined once Phase 2 (Akahu Integration) is complete.
- Candidate areas: multi-device sync, budget reporting and analytics, export to CSV/PDF, notifications.

## 🔗 Action Items

```dataviewjs
// ── Tasks for this phase only ─────────────────────────────────────────────────
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

## 📝 Spoke Notes & Documentation

- _(none yet)_

---

## ⬅️ Previous Phase

- [[03 Finance Dev - Phase 2 Akahu Integration Hub]]
