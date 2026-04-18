---
tags:
  - ChangeLog
  - HomeLabRebuild
---
# 🖥️ Homelab Change Log

> This file serves two purposes: a **manual log** for significant changes (add entries below under the correct date heading), and an **auto-timeline** at the top that pulls recent activity from phase hub files automatically.

---

## ⚡ Recent Activity — Auto Timeline

```dataviewjs
// ── Auto-generated activity timeline from homelab phase files ─────────────────
//
// Four sources of activity are merged into one chronological feed:
//
// 1. FILE MODIFICATIONS — any homelab phase file modified in the last 60 days.
//    Uses file.mtime. Proxy for "something changed in this phase".
//
// 2. FILE ADDITIONS — any homelab phase file created in the last 60 days.
//    Uses file.ctime. Distinct from modifications so new notes are visible.
//
// 3. COMPLETED TASKS — tasks with a ✅ YYYY-MM-DD completion date (Tasks plugin
//    syntax). Shows what was actually ticked off and when.
//
// 4. MANUAL LOG ENTRIES — headings in this file matching "## YYYY-MM-DD" or
//    "## DD/MM/YYYY" are parsed as manual entries and injected into the feed.
//    Write free-text entries under those headings as normal markdown.
//
// All four are sorted newest-first into a single unified timeline.

const DAYS_BACK = 60;
const cutoff    = moment().subtract(DAYS_BACK, "days");

// ── Source 1: Phase file modifications ────────────────────────────────────────
const phasePages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id === "Homelab-2025" &&
        p.phase &&
        p.file.name.includes("Hub") &&
        !p.file.folder.includes("_Archive")
    )
    .array();

const fileEvents = phasePages
    .filter(p => {
        const mtime = moment(p.file.mtime.toString());
        const ctime = moment(p.file.ctime.toString());
        // Only show as "modified" if the file was NOT created in this window
        // (new files are captured by Source 2 instead)
        return mtime.isAfter(cutoff) && !ctime.isAfter(cutoff);
    })
    .map(p => ({
        date:   moment(p.file.mtime.toString()),
        type:   "edit",
        phase:  String(p.phase),
        label:  p.file.link,
        detail: "File modified",
    }));

// ── Source 2: Newly created files ─────────────────────────────────────────────
// Pulls from ALL homelab notes (not just Hub files) so new session notes,
// stack files, and config docs are captured too.
const allHomelabPages = dv.pages('"10 - Projects"')
    .where(p =>
        p.project_id === "Homelab-2025" &&
        !p.file.folder.includes("_Archive")
    )
    .array();

const newFileEvents = allHomelabPages
    .filter(p => moment(p.file.ctime.toString()).isAfter(cutoff))
    .map(p => ({
        date:   moment(p.file.ctime.toString()),
        type:   "new",
        phase:  String(p.phase ?? "General"),
        label:  p.file.link,
        detail: "File added",
    }));

// ── Source 3: Completed tasks with dates ──────────────────────────────────────
const taskEvents = [];
for (const p of phasePages) {
    for (const t of p.file.tasks.array()) {
        if (!t.completed || !t.completion) continue;
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

// ── Source 4: Manual log entries in this file ─────────────────────────────────
// Looks for headings like "## 2026-03-15" or "## 15/03/2026" in the current file.
const manualEvents = [];
const thisFile = app.vault.getAbstractFileByPath(dv.current().file.path);
if (thisFile) {
    const raw = await app.vault.read(thisFile);
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
const allEvents = [...fileEvents, ...newFileEvents, ...taskEvents, ...manualEvents]
    .sort((a, b) => b.date - a.date);

if (allEvents.length === 0) {
    dv.paragraph(`_No homelab activity found in the last ${DAYS_BACK} days._`);
} else {
    const byDate = new Map();
    for (const ev of allEvents) {
        const key = ev.date.format("YYYY-MM-DD");
        if (!byDate.has(key)) byDate.set(key, []);
        byDate.get(key).push(ev);
    }

    const typeIcon = { edit: "📝", new: "🆕", task: "✅", manual: "📌" };

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
    .where(p => p.project_id === "Homelab-2025" && p.phase && p.file.name.includes("Hub"))
    .sort(p => String(p.phase), "asc")
    .array();

// Deduplicate by phase — collect all files per phase
const phaseMap = new Map();
const allPhaseFiles = dv.pages('"10 - Projects"')
    .where(p => p.project_id === "Homelab-2025" && p.phase)
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

        const recent = allTasks.filter(t => {
            if (!t.completed || !t.completion) return false;
            const m = moment(t.completion.toISO ? t.completion.toISO() : String(t.completion));
            return m.isValid() && m.isAfter(cutoff30);
        }).length;

        const bar = "█".repeat(Math.round(pct / 10)) + "░".repeat(10 - Math.round(pct / 10));
        return {
            done,
            remaining,
            row: [
                hub.file.link,
                `\`${bar}\` ${pct}%`,
                done,
                remaining,
                recent > 0 ? `+${recent} this month` : "—",
            ]
        };
    })
    // Filter out phases with nothing done AND nothing left — no tasks at all
    .filter(({ done, remaining }) => done > 0 || remaining > 0)
    .map(({ row }) => row);

if (rows.length === 0) {
    dv.paragraph("No homelab phase data found.");
} else {
    dv.table(["Phase", "Progress", "Done", "Left", "Recent"], rows);
}
```

---

## 📋 Manual Log Entries

> _Add dated entries below. Format: `## YYYY-MM-DD` then free text. The auto-timeline above will pick them up automatically._

---

## 2026-03-23

- [[Session Notes — DataSourceNoCloud Deep Dive & Template Fix]]

## 2026-03-19

- [[Session Notes - VM Template Debugging & Fixes]]

## 2026-03-15

- SSH fixed
- DNS fixed
- Need to get run command for docker and qemu working
- Need to start a new note or dashboard for deployed swarm nodes and services

## 2026-03-13

- Ran through Traefik again, had issues with loading
- Went back to just the template and still had issues
    - Found DNS was the root cause — updated that, then CloudInit would load
    - Now docker not installing — next step to investigate

## 2026-03-08

- New base VM template created — ran successfully after manually uploading ISO image
- Created a new traefik-worker
    - Issues with SSH and networking to be investigated

## 2026-03-02

- Back to worker VM provisioning
    - Created new script base and initial test (failed)
    - Will combine existing script with new one and update for storage changes
- Updated some storage mappings
- Reviewed change log entries for 15/02/2026 and 24/02/2026

## 2026-02-24

- Traefik dynamic configuration files applied
- Prometheus to Grafana working — Traefik dashboard created
- Proxmox to InfluxDB still an issue — connection refused
- Nebula-sync setup on new server — need to correct routing

## 2026-02-23

- Reset PiHoles and back to basic setup
- Changed their VLAN to 60
- Updated pfSense to ensure DNS is going through PiHole
- Nebula sync setup on current server

## 2025-10-25

- Worked on Consul and Traefik setup for reverse proxy and KV store
- Got some of it working — next: debug and setup PiHole (x1), Portainer, Traefik, Netbox, InfluxDB, Prometheus, Grafana
- Next: get monitoring connections working into Grafana, then update Netbox importer and PiHole HA between the two Pis

## 2025-10-16

- Worked on setting up two Raspberry Pis for PiHole and syncing — in progress
- Started using Consul for Traefik HA — decided against it, needs more research
- Main issues: HA for Traefik and PiHole

## 2025-10-10

- Deployed Swarm Managers 1, 2 and 3 successfully
- Deployed Portainer on Swarm Manager 1
- Built out Swarm Monitoring scripts and deployed via Portainer onto the managers

## 2025-10-01

- Corrected NFS mounting issue — deployed successfully
- Got three manager VMs working and linked
- Deployed Portainer in HA
- Deployed monitoring stack — Prometheus replication failed
- Notes on service storage location and persistence to be documented: [[Swarm Topology]]
- Will create a new canvas for deployment workflow overview
- Will remove existing deployment and start again — inconsistent folders and mapping found

## 2025-09-28

- Updated provisioning script to add disk partitioning (was missing)
- Added SSH keys to the template
- Updated saving of swarm tokens
- Created monitoring DB server (central) and app-db server for persisting databases and config
- Added SSH keys for above
- NFS on monitoring app to DB and saved in fstab

## 2025-09-24

- Found issue with SSH user causing problems
- Led to needing a su SSH user and a non-su SSH user for provisioning
- Created a canvas for this

## 2025-09-12

- Corrected IP issue — now getting IP from VM correctly
- Added non-root user `ssh` to template for SSH
    - Updated setup to SSH on this user — needs testing

## 2025-09-11

- Updated provisioning script to better handle ZFS creation and mounting, CloudInit, and SSH
- Still an issue on boot

## 2025-08-31

- Updated Docker Swarm template
    - Adjusted ZFS disk creation
    - Worked on debugging — issue still present, further investigation needed

## 2025-08-24

- Updated Docker Swarm VM provisioning script
    - Updated CloudInit to get correct network information
    - Reset up templates — Samba on worker, monitoring on manager

## 2025-08-22

- Updated Docker Swarm VM provisioning script
    - Added multiple NFS mounts
    - ZFS storage mount
    - CloudInit improvements
- Logged task for networking issue found during VM template testing
- Updated VM provisioning checklist

## 2025-08-16

- Setup switch bonds for Proxmox ports and added VLAN tagging
- Updated Proxmox interface to match switch bonds
- Created new checklists for VM provisioning template
- Created new Swarm worker VM template
- Created new VM provisioning scripts and ran a test

## 2025-08-11

- Consolidated notes and created new master dashboard incorporating notes and task tags

## 2025-08-04

- Test template deleted and new one created
- Provisioning script updated for NFS storage linking and local/Proxmox data persistence
- Needs testing

## 2025-08-03

- Further configured VM provisioning scripts
- Corrected NAS IP issues

## 2025-07-29

- New VM Docker template created
- Two Manager Swarm VMs created with DHCP and Docker installed

## 2025-07-27

- Setup Proxmox config and storage volumes for Docker Swarm
- Created a VM for the first Swarm manager
- Need to setup network on VM and then setup Docker

## 2025-07-20

- Added two new SSDs to the server
- Created a new storage array with RAID 1
- Installed Proxmox on it and set the IP
- Ready to confirm network configuration

## 2025-07-09

- Created a LAGG for the trunk ports
- Moved NAS to VLAN 60
- Created a bonded interface for NAS
- Got a starting firewall migration plan for pfSense for the new VLANs

## 2025-06-22

- Added new pfSense rules into pfSense machine
- Added new information on storage setup
- Updated networking information on VLAN
- Updated ToDo information and next steps

## 2025-06-14

- Started creating VLAN rules in pfSense
- Documented rules

## 2025-06-13

- Container backup script in place and running
- Discord notifications setup
- Uptime Kuma setup and sending notifications

## 2025-06-10

- Started by creating backup scripts

---