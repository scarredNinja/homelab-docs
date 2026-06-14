---
type: technical-strategy
project_id: FitnessDev-2026
phase: 'Phase 1: Foundation'
tags:
  - Fitness
  - NewtonFit
  - DockerSwarm
  - Automation
  - TechnicalReference
last_updated: '2026-05-21T00:00:00.000Z'
status: Reference
---

# 📈 NewtonFit — Data Ingestion & Target Sync Strategy

This strategy document outlines the architecture, data pathways, and automation roadmaps for streamlining fitness data ingestion into **NewtonFit** and establishing seamless target alignment across your mobile applications, dashboard, and Obsidian vault.

---

## 🏗️ 1. Core Architecture Review

NewtonFit operates inside the homelab Docker Swarm on `worker-monitoring-01` in **VLAN 60** (Management/Monitoring), routing traffic via Traefik wildcard SSL at `https://newtonfit.home.purvishome.com/`.

```mermaid
flowchart TD
    subgraph Mobile Apps
        FN[FitNotes CSV Export]
        LI[LoseIt! Daily Email CSV]
        SH[Samsung Health Connect]
    end

    subgraph Automation Layers
        ST[Syncthing / Nextcloud]
        IMAP[IMAP Email Parser Node]
        TK[Tasker + Android Client]
    end

    subgraph NewtonFit Stack (VLAN 60)
        API[Node.js server.js /api/data]
        DB[(fitness_history.json)]
        UI[Glassmorphism UI Settings]
    end

    subgraph Obsidian Vault
        ON[[Physical Goals & Biomarkers.md]]
    end

    %% Ingestion Pathways
    FN -->|Auto-Sync Folder| ST -->|Local volume watch| API
    LI -->|Automated Email| IMAP -->|HTTP POST| API
    SH -->|Health Connect| TK -->|HTTP POST| API

    %% DB Storage
    API <-->|Read / Write| DB

    %% Bidirectional Target Sync
    UI -->|HTTP POST| API
    API -->|Vault Sync script| ON
```

### Storage Foundation
* **Database File**: `/mnt/docker-data/newtonfit/data/fitness_history.json`
* **Data Fields**: `workouts`, `nutrition`, `weight`, `restingHR`, `goals`, `lipidPanel`.
* **State Mutation Strategy**: All ingestions utilize standard Map/Set deduplication key structures via the server's `POST /api/data` endpoint, ensuring that data can be uploaded repeatedly without producing duplicate records.

---

## 🚀 2. Automated Data Ingestion Pipelines

To achieve a friction-free experience, each distinct mobile health source utilizes a tailored local integration pathway.

### A. FitNotes (Workouts)
FitNotes is a clean, offline workout log. It stores records locally on Android.

* **Frictionless Pathway: Auto-Folder Mirroring & File Watcher**
  1. **Device-Side Sync**: Setup **Syncthing** or **Nextcloud** on your Android device to automatically mirror your FitNotes backup/export directory to a folder on `worker-monitoring-01` mapped to the NewtonFit dataset: `/mnt/docker-data/newtonfit/imports/fitnotes/`.
  2. **Container File Watcher**: Add `chokidar` (a Node.js file-watching package) to the NewtonFit server. Configure it to watch `/app/data/imports/fitnotes/` for new `.csv` file arrivals.
  3. **Auto-Parsing Trigger**: When a new CSV appears:
     * Parse the rows (Date, Exercise, Weight, Reps).
     * Automatically invoke the internal deduplication engine.
     * Archive the processed CSV to `/app/data/imports/fitnotes/archive/` to prevent re-processing.
     * Send a local push notification or log a success event.

### B. LoseIt! (Nutrition)
LoseIt! tracks daily macros (Protein, Carbs, Fats, Calories).

* **Frictionless Pathway: IMAP Automated Email Parser**
  1. **Daily Report Trigger**: Configure LoseIt! to automatically email a daily CSV summary of your nutrition data to a dedicated local address (e.g., `loseit-ingest@home.purvishome.com`).
  2. **Parsing Sidecar**: Run a lightweight node/python microservice (or integrate directly into `server.js`) that acts as an **IMAP listener** on the local mail server.
  3. **Auto Ingest**: The listener polls/connects via IMAP, pulls down any daily LoseIt! email attachments, parses the CSV payload, and submits it to the `POST /api/data` endpoint.

### C. Samsung Health (Weight & Resting Heart Rate)
Samsung Health tracks daily physical metrics, syncing directly to Android's built-in **Health Connect** engine.

* **Frictionless Pathway: Tasker + Local HTTP POST**
  1. **Local Network Ingress**: Because NewtonFit is exposed privately via Traefik wildcard SSL within the homelab network, your mobile phone can hit it directly whenever connected to home Wi-Fi.
  2. **Tasker Task (Android)**: Create a Tasker routine that triggers daily at 11:30 PM:
     * Use a Health Connect query action (or a helper app like *Health Sync*) to fetch the last 24 hours of weight and heart rate readings.
     * Format the data into a clean JSON array:
       ```json
       {
         "weight": [{ "date": "2026-05-21", "weight": 89.4 }],
         "restingHR": [{ "date": "2026-05-21", "value": 62 }]
       }
       ```
     * Execute an **HTTP POST** request directly to `https://newtonfit.home.purvishome.com/api/data`.
  3. **Result**: Zero cloud dependencies, absolute privacy, and fully automated daily synchronization.

---

## 🎯 3. Target Alignment & Bidirectional Sync

Maintaining target alignment (e.g. keeping bench, OHP, deadlift, or bodyweight goals in sync) requires clear mechanics.

### A. Updating Targets via NewtonFit UI
To update goals directly within your personal fitness dashboard:
1. **Target Management Panel**: Build a sleek, Nord-themed "Target settings" interface (accessible via a settings modal or dashboard card).
2. **Interactive Controls**: Include inputs for:
   * **Strength Lift Targets (5x5)**: Bench, OHP, Deadlift
   * **Biometrics**: Goal Weight, Target Body Fat, Target RHR
3. **Database Write**: Submitting the form triggers a `POST` request to `/api/data` containing the `goals` object:
   ```json
   {
     "goals": {
       "weight": 88.0,
       "bodyFat": 14.5,
       "bench": 60.0
     }
   }
   ```
   The backend merges the incoming keys into the persistent JSON database, updating your dashboard graphs and target progress bars instantly.

### B. Syncing Back to Mobile Apps
* **Constraint**: Mobile apps like FitNotes, LoseIt!, and Samsung Health are locked ecosystems; they do not expose open APIs to programmatically overwrite personal goal values on your device.
* **Sync-Assist Interface**: To resolve this, NewtonFit will display a **Sync Assist Card** whenever a target is updated:
  * Shows a clean checklist comparing dashboard goals vs app settings.
  * Provides a "Copy to Clipboard" button for easy manual entry if needed.
  * Generates an optional custom export profile (e.g., a `.csv` or custom backup file) that can be imported to force-align the mobile app's targets.

---

## 📓 4. Obsidian Vault Bidirectional Synchronization

To bring your physical metrics and targets back into your Obsidian knowledge base, a background synchronizer can bridge the gap.

### A. Automatic Note Update script
Create a scheduled script (e.g. `automation/sync-obsidian-goals.ps1` in your infrastructure repository) that runs daily:
1. **Fetch Latest Metrics**: Performs a local GET request to `https://newtonfit.home.purvishome.com/api/data` to grab the current state of goals and the latest recorded lift/biometric entries.
2. **Update Obsidian note**: Parses a target Obsidian vault note — such as `Physical Goals & Biomarkers.md` — and updates its frontmatter or Dataview parameters:
   ```markdown
   ---
   bench_target: 60
   deadlift_target: 70
   current_weight: 89.4
   last_sync: 2026-05-21
   ---
   ```
3. **Index Propagation**: Ensures any cascading links (like the primary dashboard indices or VM node tables) are kept in absolute sync.

### B. Obsidian-to-NewtonFit Seeding (Optional)
If you prefer managing targets exclusively in markdown:
* The synchronization script can do a reverse read: parse the frontmatter parameters in `Physical Goals & Biomarkers.md` and `POST` them *to* the NewtonFit API, establishing Obsidian as the single source of truth for your physical thresholds!

---

## 🛠️ 5. Implementation Roadmap

### Phase 1: Dashboard Target UI (Immediate Next Step)
* [ ] Add a `Settings` tab/modal in `index.html` styled with high-fidelity glassmorphism.
* [ ] Implement a `saveTargets()` event listener in `app.js` that compiles target fields and hits `POST /api/data`.
* [ ] Add visual target indicator lines to the SVG graphs.

### Phase 2: Android Automated Sync
* [ ] Configure **Tasker** on Android to fetch resting HR and bodyweight data.
* [ ] Add local network HTTP POST actions triggered when connecting to home SSID.
* [ ] Install **Syncthing** to automate FitNotes CSV export folders.

### Phase 3: Obsidian Sync Script
* [ ] Draft a robust PowerShell automation script in `docker-swarm-home` to extract database values and write directly into Obsidian frontmatter.
* [ ] Configure a recurring Swarm/host task or Windows Task Scheduler item to execute the sync script daily.
