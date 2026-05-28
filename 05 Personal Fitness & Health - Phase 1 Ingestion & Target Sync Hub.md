---
status: Active
priority: High
due_date: 2026-06-15
project_id: FitnessDev-2026
phase: "Phase 1: Ingestion & Target Sync"
tags:
  - fitness-app
  - newtonfit
  - tasker
  - syncthing
  - automation
---

# 🏗️ Phase 1: Ingestion & Target Sync

This phase covers the implementation of a glassmorphic target configuration panel inside the NewtonFit dashboard, the setup of Syncthing automated mirrors for FitNotes CSV logs, the deployment of Samsung Health biometrics sync via Tasker/Health Connect over secure local network HTTP POSTs, and a scheduled PowerShell script to automatically bridge targets and lifts into Obsidian notes.

### 🌐 Access Addresses
*   **Local Development Server (watch mode)**: [http://localhost:8080](http://localhost:8080)
*   **Homelab Production / Tailscale Secure Ingress**: [https://newtonfit.home.purvishome.com](https://newtonfit.home.purvishome.com)

---

## 🎯 Goals

*   **Dashboard Target UI**: Build a responsive settings panel inside NewtonFit to directly mutate goals inside the persistent JSON database.
*   **FitNotes Automation**: Mirror device workout exports via Syncthing and parse incoming CSVs via a server-side file watcher.
*   **Samsung Health Ingress**: Setup Tasker to query biometrics (weight & resting HR) and execute a local network POST payload directly to the Traefik-secured host.
*   **Obsidian Integration**: Automate bidirectional syncing, writing live dashboard stats directly into Obsidian frontmatter fields.

---

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

### 📊 Dashboard Target UI

- [x] Add glassmorphic settings cog/icon to `index.html` navigation bar [priority:: 1] #UI
- [x] Create dynamic Settings Modal/Tab styled with backdrop-filter glass cards [priority:: 1] #UI
- [x] Implement reactive fields for Strength lifts (Bench, OHP, Deadlift targets) and biometrics (Goal weight, body fat) [priority:: 1] #UI
- [x] Write `saveTargets()` client-side handler in `app.js` triggering `POST /api/data` goals merge [priority:: 2] #JS
- [x] Render dynamic target markers/lines on RHR and Bodyweight SVG trend charts [priority:: 3] #UI

### 🔄 Syncthing & CSV Watching

- [ ] Migrate testing sandbox for NewtonFit to dev-node-01 (`stack-dev-newtonfit.yml`) and verify health and sync [priority:: 1] #Swarm
- [ ] Remove/clear existing production `newtonfit` stack from `worker-monitoring-01` until sandbox testing completes [priority:: 2] #Swarm
- [ ] Setup Syncthing on Android device and configure it to mirror FitNotes export path to `worker-monitoring-01` [priority:: 1] #Syncthing
- [x] Mapped imported files folder to ZFS persistence volume `/mnt/docker-data/newtonfit/imports` [priority:: 1] #Swarm
- [x] Install `chokidar` in Node.js server container to watch imports folder [priority:: 2] #Backend
- [x] Write auto-ingestion parser to parse new workout CSVs, trigger the deduplication merge, and archive files [priority:: 2] #Backend
- [x] Test the CSV import of Samsung Health weight/RHR and FitNotes training records in the local browser via drop-zones and verify dynamic trend chart rendering [priority:: 1] #Testing

### 📱 Tasker Android Integration

- [ ] Install Tasker + Health Connect plugin (or Health Sync utility) on Android [priority:: 1] #Android
- [ ] Draft a task to query daily weight logs and average resting heart rate every night at 11:30 PM [priority:: 2] #Android
- [ ] Implement an HTTP POST Action to `https://newtonfit.home.purvishome.com/api/data` utilizing local SSID triggers [priority:: 2] #Android
- [ ] Verify SSL certificate handshakes function correctly inside local VLAN networks [priority:: 3] #Android

### 📓 Obsidian Vault Automation

- [x] Create a dedicated biomarkers file `Physical Goals & Biomarkers.md` inside your personal vault [priority:: 1] #Obsidian
- [x] Write a robust PowerShell script `automation/sync-obsidian-goals.ps1` in the `docker-swarm-home` repository [priority:: 2] #Scripting
- [x] Configure the script to perform a secure GET request to the NewtonFit domain and update markdown YAML frontmatter properties [priority:: 2] #Scripting
- [ ] Schedule the PowerShell script to run daily via Windows Task Scheduler or a cron helper node [priority:: 3] #Automation

---

## 📝 Spoke Notes & Documentation

*   **Dash Reference**: [[Service - newtonfit]]
*   **Strategy Roadmap**: [[NewtonFit - Data Ingestion & Target Sync Strategy]]
*   **Project Hub**: [[05 Personal Fitness & Health App Master Hub]]
