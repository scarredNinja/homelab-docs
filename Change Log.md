---
tags:
  - ChangeLog
  - HomeLabRebuild
status: Active
phase: Master Hub
project_id: Homelab-2025
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
                `\${bar}\` \${pct}%`,
                done,
                remaining,
                recent > 0 ? `+\${recent} this month` : "—",
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

## 12/06/2026

- **InfluxDB and Pi-hole Exporter Stack Fixes** — Updated `stack-monitoring.yml` to resolve InfluxDB migration conflicts by pinning version `2.9.0`. Configured `pihole-exporter` to run inside `debian:stable-slim` (resolving dynamic binary libc issues) with a shell wrapper entrypoint that strips trailing newlines (`tr -d '\r\n'`) from `/run/secrets/pihole_password` to prevent invalid API authentication, mounting the Go binary as a read-only volume.

## 02/06/2026

- **Permanent Netplan Routing Fix on `traefik-dmz-01` (`enp6s19`)** — Confirmed and synchronized the permanent asymmetric routing fix for the public-facing Traefik VM `traefik-dmz-01`. The fix adds `dhcp4-overrides: use-routes: false` to the `enp6s19` network interface configuration (VLAN 80) in the Netplan configuration. This prevents duplicate default routes, forces outbound traffic to consistently traverse the management network interface (`eth0`, VLAN 60), resolves asymmetry issues, survives system reboots, and aligns notes across the entire Obsidian knowledge vault.

## 01/06/2026

- **Homepage Dashboard Active Widget Integration** — Synchronized active status widgets for local homelab monitoring dashboard on `worker-monitoring-01`. Corrected Portainer environment ID to `env: 5` to resolve Portainer widget API 404 connections. Resolved self-signed TLS errors globally across Next.js API requests by adding environment variable `NODE_TLS_REJECT_UNAUTHORIZED: "0"` in `stack-homepage.yml`. Integrated Uptime Kuma native status widget directly via internal Docker network (`http://uptime-kuma_uptime-kuma:3001`). Configured active Grafana widget with admin secrets dynamically mapped via new external Docker secrets `grafana_admin_user` and `grafana_admin_password` and custom environment variables, replacing static setups.

## 28/05/2026

- **Pi-hole Widget Docker Secret Integration** — Migrated the Homepage dashboard Pi-hole widget credentials to Docker Secrets (`pihole_app_password`). Updated `stack-homepage.yml` to declare and mount the secret. Replaced the hardcoded API key placeholder in `services.yaml` with `/run/secrets/pihole_app_password`. Redeployed the Homepage stack and verified container health (running Next.js 15.4.5 cleanly). Documented the widget's path-resolution limitation and planned the transition to environment variable substitution (`{{HOMEPAGE_VAR_PIHOLE_KEY}}`) for the next session.
- **Pi-hole Monitoring Integration (PR #66)** — Deployed `pihole-exporter` as a Docker Swarm service in `stack-monitoring.yml` using `ekofr/pihole-exporter:v0.4.0` pinned to `zone=monitoring`. Configured external `pihole_password` Docker secret. Configured Prometheus scrape job `pihole-exporter` and added native `prometheus-node-exporter` targets (`10.0.60.20:9100`, `10.0.60.21:9100`) on VLAN 60 for system-level metrics.
- **Prometheus TSDB Max-Bytes Flag Bug Fix** — Resolved Prometheus service crash loops by replacing the deprecated `--storage.tsdb.max-bytes=8GB` command-line flag with the supported `--storage.tsdb.retention.size=8GB` in `stack-monitoring.yml`.
- **Finance App Container Compilation Complete** — Deployed full-stack Finance app `dev-finance` Swarm stack in Portainer, successfully compiling the Next.js/Prisma container, pushing to GHCR (`ghcr.io/scarredninja/webdev:latest`), aligning lockfiles to Prisma 6.4.0, and verifying the live service at `dev-finance.home.purvishome.com`.
- **Homelab Vault Backup & Pi-hole v6 API Migration** — Migrated the homelab Obsidian vault repository remote to HTTPS and backed up the vault along with local automation scripts to GitHub. Documented the Pi-hole v6 REST API migration paths and prepared placeholder configurations.

## 22/05/2026

- **Modular Prometheus Alerting Rules & Sync Script Automation** — Successfully modularized Prometheus alerting rules into dedicated, domain-specific files (`hardware.yml`, `zfs.yml`, and `swarm.yml`) under `config/monitoring/alerts/`. Deleted monolithic `alert-rules.yml`. Updated `prometheus.yml` to glob-match rule files (`/etc/prometheus/alerts/*.yml`) and updated the monitoring stack `stack-monitoring-yml` to mount the entire rules directory to `/etc/prometheus/alerts:ro`. Modified the config sync script `copy-swarm-config.sh` to create the target folder and glob-copy the rules dynamically. Created and pushed feature branch `feature/modular-alerting-rules`, followed by successful deployment to the Swarm cluster.
- **Inter-VM SSH Setup & Script Deployment Automation** — Established secure, automated inter-VM SSH access from the Swarm Manager VM (`manager-01`, VLAN 60) to the Swarm Worker VMs (`worker-media-01` and `worker-mediamanagement-01`, VLAN 50) using the administrative `docker` SSH user. Modified `deploy-worker-scripts.sh` to dynamically configure and update `~/.ssh/config` on `manager-01` when run with the `--remote` option. Included secure `StrictHostKeyChecking accept-new` configuration for frictionless, non-interactive host key validation and verification. Created and pushed feature branch `feature/intervm-ssh-setup`.

## 21/05/2026

- **NewtonFit Fitness & Nutrition Dashboard Deployment** — Successfully built and deployed the full-stack `NewtonFit` dashboard on Proxmox-based Docker Swarm. Automated image build and publication to private GitHub Container Registry (`ghcr.io/scarredninja/newtonfit:latest`) using `build-and-push.ps1`. Created Swarm stack `stack-newtonfit.yml` in the `docker-swarm-home` repository and integrated Traefik v3 wildcard TLS labels to securely expose it locally at `https://newtonfit.home.purvishome.com` on VLAN 60. Configured persistent local ZFS volume mounting `/mnt/docker-data/newtonfit:/app/data` on `worker-monitoring-01` (pinned via `zone == monitoring` constraint) to secure the `fitness_history.json` database. Created vault documentation `Service - newtonfit.md` and updated placement matrices in `Docker Swarm Details.md`.

## 20/05/2026

- **Alertmanager Discord Alerting (PR #48)** — Deployed Alertmanager on the dedicated `worker-monitoring-01` VM (`prom/alertmanager:v0.27.0`), pinned to `zone=monitoring`. Created alert rules in `config/monitoring/alert-rules.yml` and configured Discord webhook alerts with successful notification testing. Added Traefik dynamic routing to expose Alertmanager at `https://alertmanager.home.purvishome.com` and configured DNS A record. Added pfSense rule allowing VLAN 60 to VLAN 50 TCP/22 SSH access for remote management.
- **Exportarr Sidecars & compose-vpn Update (PR #49)** — Resolved critical Exportarr startup empty `URL` validation crashes by configuring Sonarr and Radarr sidecars to use standard, prefix-free environment variables (`PORT`, `URL`, `API_KEY_FILE`). Resolved the `regex: api-key must be a 20-32 character alphanumeric string` crash loop caused by trailing newlines in Docker secrets by deploying clean `sonarr_apikey_v2` and `radarr_apikey_v2` secrets created via `printf`.
- **Prometheus Scrape IP Correction** — Corrected `exportarr-sonarr` and `exportarr-radarr` scrape targets in `prometheus.yml` from `10.0.50.30` to `10.0.50.51` (the actual IP address of `worker-mediamanagement-01`), enabling successful metrics collection. Confirmed both targets are up and scraping on ports `9707` and `9708`.
- **compose-vpn Service Fixes** — Modified `compose-vpn.service` on `worker-mediamanagement-01` to add a robust container cleanup hook (`docker rm -f gluetun`) to ensure smooth restarts without manual intervention. Verified Gluetun successfully running on WireGuard (NordLynx) and Transmission running smoothly.

## 08/05/2026

- Plex media mount fix — container saw empty `/media/Movies` etc. despite host having NFS shares mounted. Root cause: stack bind-mounted parent `/mnt/media`, but host uses **autofs direct-mount triggers** per share — Docker default `rprivate` bind propagation captures trigger directories but NOT the underlying NFS mounts. Fix: bind each share directly (`/mnt/media/Movies:/media/Movies:ro`, etc.). PR [#30](https://github.com/scarredNinja/docker-swarm-home/pull/30), commit `ffd7d90`. Container paths unchanged — no Plex library reconfiguration needed. Added Gotcha #60.
- Stale autofs trigger `/mnt/media/tvshows` (lowercase) flagged for cleanup — non-blocking.
- `worker-mediamanagement-01` vCPU 4 → 6 — Radarr/Sonarr UI was unresponsive during heavy Transmission sessions. Root cause: gluetun's OpenVPN is single-threaded and pins a full vCPU at 100% under load (AES-NI confirmed available — OpenVPN protocol limit, not crypto). Radarr was CPU-starved despite using <1% itself. Bump to 6 cores leaves OpenVPN/transmission their dedicated core(s) and frees the rest for the Arr stack. Proper fix queued: migrate gluetun to WireGuard (NordLynx).
- cloudflared tunnel token persistence fix — `plex_cloudflared` was 0/1 with empty `--token` arg; root cause was `${CLOUDFLARE_TUNNEL_TOKEN}` interpolating from operator shell (unset on fresh ssh / reboot)
- Solution: persist token in Portainer stack env vars (Stacks → plex → Environment variables); Portainer always interpolates at deploy time regardless of shell state
- Discovered `cloudflare/cloudflared:latest` is **distroless** — no `/bin/sh`, no debug tools. Entrypoint shim approach (Docker secret + `sh -c 'exec cloudflared ...'`) impossible. PRs [#26](https://github.com/scarredNinja/docker-swarm-home/pull/26) + [#27](https://github.com/scarredNinja/docker-swarm-home/pull/27) reverted via [#28](https://github.com/scarredNinja/docker-swarm-home/pull/28)
- Stale `cf_tunnel_token` Docker secret can be removed: `docker secret rm cf_tunnel_token`
- Verified: 1/1 replicas, 4× "Registered tunnel connection"
- Added Gotcha #59 to runbook
- compose-vpn (gluetun + transmission) auto-start fix — `restart: unless-stopped` was failing post-reboot due to Swarm overlay gossip race (Gotcha #58)
- New systemd unit `compose-vpn.service`: oneshot, waits for `Swarm.LocalNodeState=active` + probe-attaches to `arr_arr_default` and `traefik-public` before running `docker compose up -d` (5× retry, `-p compose-vpn` pins project name)
- Generalized provisioning path for future plain-compose stacks: `02-provision-vm.sh --compose-unit <name>` writes registry; `03-post-boot.sh install_compose_units` copies unit + compose file from virtiofs and `systemctl enable`s (no auto-start — env files must land first); `copy-swarm-config.sh` syncs `proxmox-swarm/systemd/*.service` to `/mnt/docker-swarm/systemd/`
- Added `UPDATER_PERIOD: 24h` to gluetun (stale NordVPN server list surfaces as TLS handshake / EHOSTUNREACH)
- Verified post-reboot: unit `active (exited)`, gluetun + transmission Up, exit IP `180.149.231.200` (Auckland NordVPN)
- PR [#25](https://github.com/scarredNinja/docker-swarm-home/pull/25), branch `claude/mystifying-engelbart-998246` — commits `c8f4b2c`, `eb9e5cf`, `5ecce9b`

---

### 29/04/2026

- UniFi AP re-adopted to new controller on VLAN 60 (factory reset required — CGNAT blocks Ubiquiti cloud adoption relay, Bug #48)
- Remote Access disabled on UniFi controller — local-only adoption going forward
- SSIDs confirmed: Netbacon (VLAN 10), IoT (VLAN 20), Guest (VLAN 30)
- unifi-poller deployed — PR #14 merged (3 commits)
- Traefik middleware reference bug fixed: `internal-only` → `internal-only@file` (Bug #49)
- Deploy script updated: middlewares.yaml and vpn-transmission.yml now copied
- Open: unifi.home.purvishome.com 404, unifi-poller scrape not yet verified

---

## 28/04/2026

- Fixed Transmission file deletion — chown migrated downloads from UID 1001 → 1000
- Fixed HA bad gateway — added port 8123 host-mode publish + trusted_proxies config
- Traefik dynamic config refactored — infrastructure.yml split into per-service files
- UniFi Traefik labels fixed — added internal-only middleware + insecureTransport
- Prometheus HA scrape job added — bearer token stored as Docker secret
- Redback solar integration confirmed working post-migration
- PR #13 merged — traefik dynamic config split
- Added Gotchas #45, #46, #47 to runbook Appendix D
- Added runbook task: migrate stack deployments to Git repo via Portainer

## 27/04/2026

- InfluxDB token fixed post-migration, v1 compatibility configured (DBRP + v1 auth)
- Grafana datasources updated: InfluxQL + Flux, both Save & Test passing
- Monitoring stack: InfluxDB added to traefik-public network (Bug #43)
- Traefik metrics fixed: insecure: true + entryPoint: traefik + rogue config removed (Bug #44)
- Transmission + Gluetun deployed via compose-vpn.yml on worker-mediamanagement-01
- Arr stack deployed: Sonarr, Radarr, Prowlarr all 1/1, imports in progress
- Downloads migration completed: subvol-110 destroyed
- MainStorage stale VM disks destroyed: vm-100, vm-106, vm-108 (~208G recovered)
- MainStorage/downloads quota set: 500G
- MainStorage/backups created: 150G reservation

## 25/04/2026

- Fixed Portainer agent overlay: deleted stale VXLAN interfaces on manager-01, cleared `local-kv.db` on all nodes, force-updated portainer agent service
- Configured inter-VM SSH from manager-01 to all workers (`~/.ssh/id_ed25519`)
- Split Transmission + Gluetun out of Swarm arr stack → Docker Compose (`compose-vpn.yml`) on worker-mediamanagement-01
- Made `arr_arr_default` overlay attachable so compose containers can join Traefik routing
- Added Traefik file-provider route `vpn-transmission.yml` for Transmission
- Fixed YAML parse error in `infrastructure.yml` (missing colon on `homeassistant-service`) — was silently breaking all file-based Traefik routes
- Switched Gluetun VPN: Mullvad WireGuard → NordVPN OpenVPN (WireGuard key registration blocked by Cloudflare)
- Fixed `acme.json` permissions 755 → 600; Traefik cert resolver Cloudflare now active
- Updated Sonarr/Radarr download client host: `transmission` → `gluetun`
- PR #10 merged to main

## 24/04/2026

- Final service migration from MainStorage LXC containers to Docker Swarm VMs
- Provisioned `worker-controller-01` (10.0.60.42, VLAN 60+20)
- Migrated: Sonarr, Radarr, Prowlarr, Transmission config, Plex (delta), Tautulli, InfluxDB, Home Assistant, UniFi, Uptime Kuma, Prometheus (5.4GB solar history), Grafana (dashboards preserved)
- Downloads transfer (440GB) in progress
- Fixed 3 stack bugs: Gluetun Swarm incompatibility (Bug #26), virtiofs permissions post-rsync (Bug #27), UniFi image MongoDB requirement (Bug #28)
- Stack changes: arr (Gluetun removed, Transmission fixed), controller (new), uptime-kuma (new), plex (ADVERTISE_IP, dual routers)
- PRs merged: #8 (arr fix), #9 (controller stack)

## 2026-03-23

- [[Session Notes — 2026-03-23 — DataSourceNoCloud Template Fix]]

## 2026-03-19

- [[Session Notes — 2026-03-19 — VM Template Debugging]]

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
