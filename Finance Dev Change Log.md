# 💰 Finance Dev Change Log

> This file serves two purposes: a **manual log** for significant changes (add entries below under the correct date heading), and an **auto-timeline** at the top that pulls recent activity from phase hub files automatically.

---

## ⚡ Recent Activity — Auto Timeline

```dataviewjs
// ── Auto-generated activity timeline from Finance Dev phase files ─────────────
//
// Three sources of activity are merged into one chronological feed:
//
// 1. FILE MODIFICATIONS — any Finance Dev phase file modified in the last 60 days.
//    Proxy for "something changed in this phase". Not granular but always works.
//
// 2. COMPLETED TASKS — tasks with a ✅ YYYY-MM-DD completion date (Tasks plugin
//    syntax). Shows what was actually ticked off and when. Only appears if you
//    use the Tasks plugin completion date format.
//
// 3. MANUAL LOG ENTRIES — headings in this file matching "## YYYY-MM-DD" or
//    "## DD/MM/YYYY" are parsed as manual entries and injected into the feed.
//    Write free-text entries under those headings as normal markdown.
//
// All three are sorted newest-first into a single unified timeline.

const DAYS_BACK = 60;
const cutoff    = moment().subtract(DAYS_BACK, "days");

// ── Source 1: Phase file modifications ────────────────────────────────────────
const phasePages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id === "FinanceDev-2026" &&
        p.phase &&
        p.file.name.includes("Hub") &&
        !p.file.folder.includes("_Archive")
    )
    .array();

const fileEvents = phasePages
    .filter(p => moment(p.file.mtime.toString()).isAfter(cutoff))
    .map(p => ({
        date:   moment(p.file.mtime.toString()),
        type:   "edit",
        phase:  String(p.phase),
        label:  p.file.link,
        detail: `File modified`,
    }));

// ── Source 2: Completed tasks with dates ──────────────────────────────────────
const taskEvents = [];
for (const p of phasePages) {
    for (const t of p.file.tasks.array()) {
        if (!t.completed || !t.completion) continue;
        // t.completion from Dataview is a Luxon DateTime
        const completedMoment = moment(t.completion.toISO ? t.completion.toISO() : String(t.completion));
        if (!completedMoment.isValid() || completedMoment.isBefore(cutoff)) continue;
        taskEvents.push({
            date:   completedMoment,
            type:   "task",
            phase:  String(p.phase),
            label:  p.file.link,
            detail: t.text.replace(/\s*✅\s*\d{4}-\d{2}-\d{2}/, "").trim(),
        });
    }
}

// ── Source 3: Manual log entries in this file ─────────────────────────────────
// Looks for headings like "## 2026-04-01" or "## 01/04/2026" in the current file.
// The content under each heading is shown as the detail.
const manualEvents = [];
const thisFile = app.vault.getAbstractFileByPath(dv.current().file.path);
if (thisFile) {
    const raw = await app.vault.read(thisFile);
    // Match ## YYYY-MM-DD or ## DD/MM/YYYY headings and capture text until next ##
    const logPattern = /^##\s+(\d{4}-\d{2}-\d{2}|\d{2}\/\d{2}\/\d{4})\s*[^\n]*\n([\s\S]*?)(?=^##|\Z)/gm;
    let match;
    while ((match = logPattern.exec(raw)) !== null) {
        const rawDate = match[1];
        const parsed  = moment(rawDate, ["YYYY-MM-DD", "DD/MM/YYYY"], true);
        if (!parsed.isValid() || parsed.isBefore(cutoff)) continue;
        const detail = match[2].trim().replace(/\n+/g, " · ").slice(0, 200);
        if (!detail) continue;
        manualEvents.push({
            date:   parsed,
            type:   "manual",
            phase:  "Manual entry",
            label:  null,
            detail,
        });
    }
}

// ── Merge and sort all events newest-first ────────────────────────────────────
const allEvents = [...fileEvents, ...taskEvents, ...manualEvents]
    .sort((a, b) => b.date - a.date);

if (allEvents.length === 0) {
    dv.paragraph(`_No Finance Dev activity found in the last ${DAYS_BACK} days._`);
} else {
    // Group by date for a cleaner timeline view
    const byDate = new Map();
    for (const ev of allEvents) {
        const key = ev.date.format("YYYY-MM-DD");
        if (!byDate.has(key)) byDate.set(key, []);
        byDate.get(key).push(ev);
    }

    const typeIcon = { edit: "📝", task: "✅", manual: "📌" };

    for (const [dateKey, events] of byDate) {
        const dateLabel = moment(dateKey).format("ddd D MMM YYYY");
        dv.el("div", `**${dateLabel}**`, {
            attr: { style: "font-weight:700;color:#88c0d0;margin-top:12px;margin-bottom:2px;border-bottom:1px solid rgba(136,192,208,0.2);padding-bottom:2px" }
        });

        for (const ev of events) {
            const icon     = typeIcon[ev.type] ?? "•";
            const phaseTag = `<span style="font-size:0.78em;color:#6b7a8a;margin-left:4px">${ev.phase}</span>`;
            const linkPart = ev.label ? `${ev.label} ` : "";
            const detail   = ev.detail ? `<span style="color:#d8dee9"> — ${ev.detail}</span>` : "";
            dv.el("div", `${icon} ${linkPart}${detail}${phaseTag}`, {
                attr: { style: "font-size:0.85em;margin:2px 0 2px 8px;line-height:1.5" }
            });
        }
    }
    dv.el("div", `_Showing last ${DAYS_BACK} days · ${allEvents.length} events_`, {
        attr: { style: "font-size:0.78em;color:#4a5568;margin-top:12px" }
    });
}
```

---

## ✅ Task Completion Stats

```dataviewjs
// ── How many tasks completed per phase, total and in last 30 days ─────────────
const cutoff30 = moment().subtract(30, "days");

const phases = dv.pages('"10 - Projects"')
    .where(p => p.project_id === "FinanceDev-2026" && p.phase && p.file.name.includes("Hub"))
    .sort(p => String(p.phase), "asc")
    .array();

// Deduplicate by phase — collect all files per phase
const phaseMap = new Map();
const allPhaseFiles = dv.pages('"10 - Projects"')
    .where(p => p.project_id === "FinanceDev-2026" && p.phase)
    .array();

for (const p of allPhaseFiles) {
    const key = String(p.phase);
    if (!phaseMap.has(key)) phaseMap.set(key, { hub: null, files: [] });
    const entry = phaseMap.get(key);
    entry.files.push(p);
    if (!entry.hub || p.file.name.includes("Hub")) entry.hub = p;
}

const rows = [...phaseMap.entries()]
    .sort(([a], [b]) => a.localeCompare(b, undefined, { numeric: true }))
    .map(([phaseLabel, { hub, files }]) => {
        const allTasks   = files.flatMap(f => f.file.tasks.array());
        const total      = allTasks.length;
        const done       = allTasks.filter(t => t.completed).length;
        const remaining  = total - done;
        const pct        = total > 0 ? Math.round((done / total) * 100) : 0;

        // Recent completions: tasks completed in last 30 days
        const recent = allTasks.filter(t => {
            if (!t.completed || !t.completion) return false;
            const m = moment(t.completion.toISO ? t.completion.toISO() : String(t.completion));
            return m.isValid() && m.isAfter(cutoff30);
        }).length;

        const bar    = "█".repeat(Math.round(pct / 10)) + "░".repeat(10 - Math.round(pct / 10));
        return [
            hub.file.link,
            `\`${bar}\` ${pct}%`,
            done,
            remaining,
            recent > 0 ? `+${recent} this month` : "—",
        ];
    });

if (rows.length === 0) {
    dv.paragraph("No Finance Dev phase data found.");
} else {
    dv.table(["Phase", "Progress", "Done", "Left", "Recent"], rows);
}
```

---

## 📋 Manual Log Entries

> _Add dated entries below. Format: `## YYYY-MM-DD` then free text. The auto-timeline above will pick them up automatically._

---

## 2026-04-01

- Planned Phase 1 (Foundation) and Phase 2 (Akahu Integration) — specs documented
- Created Master Hub, Phase 1 Hub, Phase 2 Hub vault notes
- Documented Akahu integration architecture and pseudocode — [[Akahu Integration - Architecture & Pseudocode]]
- App deployed and running at `http://localhost:3000` via Docker
- Existing features: budget envelopes, transactions, savings goals, mortgage calculator, solar, property valuation

---