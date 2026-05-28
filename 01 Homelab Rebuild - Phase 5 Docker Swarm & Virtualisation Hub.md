---
status: Complete
priority: High
due_date: 2026-04-15
project_id: Homelab-2025
phase: "Phase 5: Docker Swarm"
---
# 🔄 Phase 5: Docker Swarm & Virtualisation

Setting up the container orchestration platform to host your services, focusing on stability and scale.

## 🎯 Goals

* Deploy all 5 Swarm nodes (1 manager, 4 workers)
* ✅ Deploy Traefik to DMZ node and verify HTTPS — complete 2026-04-21
* ✅ Plex external access — DNS configured, browser test next session
* Migrate legacy services from old Compose server via ZFS send/receive
* Verify cluster stability before Phase 6

---

## 🖥️ VM Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-vm")
  .sort(p => p.vmid ?? 999, "asc");

const statusIcon = s => ({
  "not-created": "⬜",
  "provisioned": "🟡",
  "post-boot-complete": "🟠",
  "running": "🟢",
}[s] ?? "❓");

dv.table(
  ["VM", "VMID", "VLAN(s)", "IP", "Status", "Post-Boot", "Swarm"],
  pages.map(p => [
    p.file.link,
    p.vmid ?? "—",
    [p.vlan_primary, p.vlan_secondary].filter(Boolean).join(" + "),
    p.ip_primary ?? "—",
    statusIcon(p.vm_status) + " " + (p.vm_status ?? "—"),
    p.post_boot_run ? "✅" : "🔲",
    p.swarm_joined ? "✅" : "🔲",
  ])
);
```

---

## 🐳 Service Status

```dataviewjs
const pages = dv.pages('"10 - Projects"')
  .where(p => p.type === "swarm-service")
  .sort(p => p.service_name, "asc");

const statusIcon = s => ({
  "pending":   "⬜",
  "deploying": "🟡",
  "deployed":  "🟢",
  "running":   "🟢",
  "degraded":  "🔴",
}[s] ?? "❓");

dv.table(
  ["Service", "VM", "Status", "Port", "External", "Dataset"],
  pages.map(p => [
    p.file.link,
    p.vm ?? "—",
    statusIcon(p.service_status) + " " + (p.service_status ?? "—"),
    p.port ?? "—",
    p.external_access ? "🌐 Yes" : "🔒 No",
    p.zfs_dataset ?? "—",
  ])
);
```

---

## 🔗 Open Tasks



```dataviewjs
const PRIORITY_FALLBACK = 999;
const thisPhase = dv.current().phase;

const pages = dv.pages('"10 - Projects"')
  .where(p =>
    p.phase === thisPhase &&
    !p.file.folder.includes("_Archive") &&
    p.type !== "swarm-vm" &&
    p.type !== "swarm-service"
  );

const DEFERRED = ["#backlog", "#Later", "#MuchLater"];
const tasks = pages
  .flatMap(p => p.file.tasks
    .where(t => !t.completed && !DEFERRED.some(tag => (t.tags ?? []).includes(tag)))
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
```

> [!note] Ordered work lives in [[Docker Swarm Infrastructure Runbook]] phase checklists.
> Current blockers and next actions: [[Swarm Topology]]

> [!abstract]- 📦 Backlog — action later
>
> ```dataviewjs
> const PRIORITY_FALLBACK = 999;
> const DEFERRED = ["#backlog", "#Later", "#MuchLater"];
> const thisPhase = dv.current().phase;
>
> const pages = dv.pages('"10 - Projects"')
>   .where(p =>
>     p.phase === thisPhase &&
>     !p.file.folder.includes("_Archive") &&
>     p.type !== "swarm-vm" &&
>     p.type !== "swarm-service"
>   );
>
> const tasks = pages
>   .flatMap(p => p.file.tasks
>     .where(t => !t.completed && DEFERRED.some(tag => (t.tags ?? []).includes(tag)))
>     .map(t => ({ task: t, file: p.file }))
>   )
>   .array();
>
> if (tasks.length === 0) {
>   dv.paragraph("✅ No backlog tasks.");
> } else {
>   tasks.sort((a, b) => {
>     const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
>     const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
>     if (pa !== pb) return pa - pb;
>     if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
>     return (a.task.line ?? 0) - (b.task.line ?? 0);
>   });
>   dv.taskList(tasks.map(t => t.task), false);
> }
> ```

---


- [x] Fix Obsidian Bases files — Swarm VM Board and Swarm Service Table parsing errors [priority:: 2]
	> The DataviewJS queries above (VM Status and Service Status tables) query on `type == "swarm-vm"` and `type == "swarm-service"`. If the tables are empty or erroring, verify that the individual VM/service notes (e.g. `Service - traefik.md`) have `type: swarm-vm` or `type: swarm-service` set correctly in their frontmatter. The Obsidian Bases board is a separate view — check that the Bases config file points to `"10 - Projects"` and filters on the same `type` field.
  
- [x] Summerise code session and also check if PBS will even be needed. Also look to add in dashboard to show backup information and status
- [x] Run speed checks on file loads from Nas to vm for media. Create a script to handle this for future use. Check prometheus stats issue [priority::1] ✅ 2026-05-19 — nfs-bench.sh deployed via PR #40; Prometheus VLAN 50 external scrape targets added; deploy-worker-scripts.sh automates cron deploy to worker-media-01
- [x] Run nfs-bench.sh on worker-mediamanagement-01 for second data point [priority::2] ✅ 2026-05-22 — deployed via deploy-worker-scripts.sh; baseline and load results analyzed in [[NFS Mount Performance Analysis]]
- [x] Update gluetan/transmission information on how the api key for exportarr is now configured. Update api key setting name to api key file entry. https://github.com/onedr0p/exportarr [priority::1] ✅ 2026-05-20
- [x] Investigate Radarr/Sonarr UI load latency on `worker-mediamanagement-01` — root cause (OpenVPN CPU pin) eliminated via WireGuard migration ✅ 2026-05-17. Post-redeploy UI latency still under investigation.
- [x] Deploy docker service to sync pihole settings ✅ 2026-05-24

## 📝 Reference Docs

### Architecture
- [[Swarm Topology]] ← living dashboard, current blockers and next actions
- [[Docker Swarm Infrastructure Runbook]] ← ordered phase checklists
- [[Traefik Routing Architecture]] ← entrypoints, service split, VPN access model
- [[Docker Swarm Details]] ← service placement and storage matrix
- [[ZFS Configuration and Setup]]
- [[VLAN and Subnet Summary Sheet]]

### Monitoring
- [[Monitoring Docker Swarm]]
- [[z_Archived-Docker Swarm Monitoring]]

### Networking
- [[Netbox Setup]]
- [[Docker Swarm Scaling Strategy]]

---

## 📋 Working Sessions (chronological)

| Session | Date | Key outcome |
| ------- | ---- | ----------- |
| [[Session Notes — 2026-03-19 — VM Template Debugging]] | 2026-03-19 | SSH key setup, template bugs, overlay2→fuse-overlayfs |
| [[Session Notes — 2026-03-23 — DataSourceNoCloud Template Fix]] | 2026-03-23 | NBD fix for 99-pve.cfg, DataSourceNoCloud root cause |
| [[Session Notes — 2026-03-24 — Multi-NIC Netplan virtiofs]] | 2026-03-24 | Interface naming, multi-NIC, netplan timing |
| [[Session Notes — 2026-03-24 — Multi-NIC Netplan virtiofs]] | 2026-03-27 | virtiofs mapping ID fix, multi-NIC support, pfSense DHCP prompt |
| [[Session Notes — 2026-03-27 — virtiofs Multi-NIC Provision Flow]] | 2026-03-27 | virtiofs provision flow, multi-NIC provisioning |
| [[Session Notes — 2026-03-27 — Provisioning Scripts v2 Complete]] | 2026-03-27 | Scripts v2 complete, ZFS zvol backup gap identified |
| [[Session Notes — 2026-03-27 — Provisioning Scripts Live Debug]] | 2026-03-27 | virtiofs syntax, idx++ bug, qm agent jq fix, VLAN 60 routing |
| [[Session Notes — 2026-03-30 — Post-Boot ZFS Tuning Traefik Stack]] | 2026-03-30 | 03-post-boot v2, ZFS tuning scripts, Traefik v3 stack |
| [[Session Notes — 2026-03-31 — Docker Swarm Pipeline Fixes]] | 2026-03-31 | Pipeline bugs fixed, Portainer Swarm mode, manager-01 + traefik-dmz-01 provisioned |
| [[Session Notes — 2026-04-02 — virtiofs Permissions Provisioning Bugs]] | 2026-04-02 | daemon.json path bug, fstab missing, ZFS permissions, DHCP blocker |
| [[Session Notes — 2026-04-05 — Phase 0 Swarm Nodes Portainer]] | 2026-04-05 | Finished most of Phase 0 (beside PBS), deployed manager and traefik VMs |
| [[Session Notes — 2026-04-07 — Traefik Config Docker Fixes]] | 2026-04-07 | Worked on and fixed Traefik configuration files with Claude Code |
| [[Session Notes — 2026-04-08 — Traefik Deployment]] | 2026-04-08 | Docker Swarm issues and Traefik deployment |
| [[Session Notes — 2026-04-08 — Swarm Corrections]] | 2026-04-08 | Swarm corrections and Traefik deployment fixes |
| [[Session Notes — 2026-04-10 — Traefik + Portainer Deployment]] | 2026-04-10 | Swarm manager recovery, virtiofs permission fixes, full Traefik + Portainer deploy with wildcard TLS |
| [[Session Notes — 2026-04-12 — Swarm Init Fix]] | 2026-04-12 | Swarm init error fixed, Portainer + Traefik redeployed, Traefik issues diagnosed |
| [[Session Notes — 2026-04-12 — Monitoring Deploy]] | 2026-04-12 | Docker install corrected, scripts updated, monitoring stack deployed |
| [[Session Notes — 2026-04-18 — Monitoring Complete Media Planning]] | 2026-04-18 | Monitoring finished, media VM architecture decisions made |
| [[Session Notes — 2026-04-18 — Media VM Provision]] | 2026-04-18 | Provisioned media and mediamanagement VMs, copied media files |
| [[Session Notes — 2026-04-19 — Plex Arr Stack Deploy]] | 2026-04-19 | Plex and Arr stacks deployed, Traefik cert resolver fixed |
| [[Session Notes — 2026-04-21]] | 2026-04-21 | Fixed HTTPS (asymmetric routing Bug #21 + overlay subnet collision Bug #22). Overlay subnets pinned to 10.200.x.0/24. Plex DNS configured. |
| [[Session Notes — 2026-04-23 — Plex External Access & Sonarr Fix]] | 2026-04-23 | CGNAT discovered, Cloudflare Tunnel deployed, Plex external working, Sonarr 404 fixed, VXLAN inter-VLAN firewall rules added |
| [[Session Notes — 2026-04-24 — Final Service Migration & Stack Deployment]] | 2026-04-24 | All service configs migrated from MainStorage LXC containers, worker-controller-01 provisioned, arr/plex/controller/uptime-kuma stacks fixed and deployed, 3 stack bugs resolved |
| [[Session Notes — 2026-04-25 — Portainer Overlay Fix & Transmission VPN Split]] | 2026-04-25 | Stale VXLAN + local-kv.db overlay fix, inter-VM SSH configured, Transmission+Gluetun split to Docker Compose, arr_arr_default made attachable, Traefik file-provider YAML fix, acme.json permissions fixed |
| [[Session Notes — 2026-04-26 — rpool Full Recovery & Downloads Migration]] | 2026-04-26 | rpool recovered (577G free), downloads migrated SSD→MainStorage HDD, old LXC subvolumes cleared, Traefik entrypoint fix planned |
| [[Session Notes — 2026-04-27 — Monitoring Recovery, Arr Stack & Storage Cleanup]] | 2026-04-27 | InfluxDB token fixed + v1 compat, Grafana datasources configured, Traefik metrics fixed, Transmission + arr stack deployed, MainStorage cleaned up and quota set |
| [[Session Notes — 2026-04-28 — HA Routing Fix & Traefik Config Split]] | 2026-04-28 | HA accessible, Traefik dynamic config refactored, UniFi labels fixed, Prometheus HA scrape active |
| [[Session Notes — 2026-04-29 — UniFi Adoption & unifi-poller Setup]] | 2026-04-29 | AP re-adopted to VLAN 60, unifi-poller deployed via PR #14, 2 bugs found |
| [[Session Notes — 2026-04-30 — Backup Scripts & Uptime Kuma]] | 2026-04-30 | ZFS snapshot fixed, proxmox config backup deployed, restic scaffolded (blocked), Uptime Kuma PR #16 merged |
| [[Session Notes — 2026-05-08 — Plex Media Mount Fix]]<br>[[Session Notes — 2026-05-13 — Backup Script Fixes & Disk Monitoring]] | 2026-05-11 | Backup script bugs fixed (PR #31), Plex DNS/routing fixed, transcode settings optimised, CPU type set to host. *(Note: The original 2026-05-11 session note was never created).* |
| [[Session Notes — 2026-05-13 — Backup Script Fixes & Disk Monitoring]] | 2026-05-13 | restic --fail preflight fixed (184 GiB backed up), vm-backup.sh PATH fixed (6/6 VMs), disk-space-check.sh deployed, Grafana storage row, notification consolidation |
| [[Session Notes — 2026-05-17 — WireGuard Migration & Exportarr]] | 2026-05-17 | Gluetun migrated to WireGuard (NordLynx), root cause of UI latency eliminated, Exportarr sidecars added, arr stack network/port fixes, Gotcha #70, PR #39 |
| [[Session Notes — 2026-05-19 — Monitoring Network Stack Consolidation]] | 2026-05-19 | Monitoring overlay decoupled (PR #41), NFS bench + VLAN 50 scrape targets (PR #40), restic deferred sentinel (PR #38), WireGuard migration + Exportarr sidecars (PR #39), stack consolidation (PR #42/#43), automation fix (PR #44) |
| [[Session Notes — 2026-05-20 — Alertmanager Discord Alerting]] | 2026-05-20 | Deployed Alertmanager with Discord webhook alerting, tested notifications, configured inter-VLAN SSH rules (PR #48) |
| [[Session Notes — 2026-05-20 — Exportarr & compose-vpn Update]] | 2026-05-20 | Fixed Exportarr sidecar configurations (prefix-free env vars, `_v2` secrets created via `printf`) in `stack-arr.yml`, audited and verified standalone `compose-vpn.yml` systemd service, corrected Prometheus scrape targets from `.50.30` to `.51`, and verified metrics collection is UP and running cleanly. |
| [[Session Notes — 2026-05-21 — Uptime Kuma Deployment]] | 2026-05-21 | PR #49 fixed Uptime Kuma Traefik routing (correct hostname, `traefik.swarm.network`, `internal-only@file`); stack redeployed and confirmed working at `https://uptime-kuma.home.purvishome.com`. Homepage dashboard deferred. |
| [[Session Notes — 2026-05-22 — Restic NFS Fix & Homepage Dashboard]] | 2026-05-22 | PR #50 merged (NFS soft mount fix for restic); stale restic lock cleared on Proxmox host; backup verified working. PR #23 merged (Homepage dashboard, `stack-homepage.yml`); hostname `dashboard.home.purvishome.com`. PR #31 closed (superseded). PRs #45/#46/#47/#48/#49 confirmed merged. |
| [[Session Notes — 2026-05-24 — dev-node-01 Provision & MCP Deploy]] | 2026-05-24 | Provisioned dev-node-01 (VMID 206, VLAN 40, 10.0.40.50); fixed cloud-init DNS stall + SSH VLAN routing issue; node joined swarm (7 nodes total). Obsidian MCP repo strategy decided (dedicated public repo). `stack-obsidian-mcp.yml` + `stack-syncthing.yml` authored and deployed — both at 0/1 pending bind mount creation on dev-node-01. |
| [[Session Notes — 2026-05-25 — dev-node-01 Provision & MCP Deploy]] | 2026-05-25 | Fixed 4 sequential blockers (bind mounts, ghcr.io image, acme.json 600 perms, docker-proxy restart policy); both services at 1/1. Syncthing peered with Windows client. Vault synced. Diagnosed + fixed `VAULT_ALLOWED_FOLDERS` path mismatch (files sync flat, not under subdirectory). MCP read fully verified. Fixed Homepage healthcheck IPv6 failure (`127.0.0.1`). Session 4: diagnosed + fixed `VAULT_WRITE_FOLDERS` EACCES bug (PRs #55/#56); MCP write confirmed end-to-end. Session 5: updated agent definitions (`obsidian-search.md`, `obsidian-editor.md`, `session-note-writer.md`) to use MCP tools natively. Session 6: added `vault-sync` and `changelog-writer` agents; added `scheduled-changelog.ps1` and `scheduled-progress.ps1` automation scripts. Session 7: resolved `client-bridge.ts` stateful session-tracking, empty notification body, and Windows console logging stability bugs; parallelized backlinks lookups in `src/services/vault.ts` (20x speedup); migrated 7 custom agents into global Antigravity config directory. |
| [[Session Notes — 2026-05-25 — Swarm Manager HA & Backup Automation]] | 2026-05-25 | Provisioned manager-02 (VMID 210, 10.0.60.31) and manager-03 (VMID 211, 10.0.60.32) — Swarm now has 3-manager quorum (Leader + 2 Reachable). Fixed `(( ++ ))` arithmetic bug in `02-provision-vm.sh` (PR #57). Refactored `vm-backup.sh` to registry-driven VM discovery — no more hardcoded VMIDs, new VMs auto-included after provision (PR #58). Overnight backup run pending to verify all 8 VMs complete within 2h window. |
| [[Session Notes — 2026-05-26 — Security Hardening & Secrets Migration]] | 2026-05-26 | Pinned all :latest image tags to stable versions (PR #61). Migrated Grafana admin creds, NetBox DB/secret key, cloudflared tunnel token, and restic REST auth to Docker secrets. Restic `--no-auth` replaced with `--htpasswd-file`. Prometheus interval 15s→30s + 8GB TSDB cap. Traefik compress middleware added. Fixed uptime-kuma Homepage widget slug (PR #59). |
| [[Session Notes — 2026-05-27 — Dev Sandboxes & Ingestion Setup]] | 2026-05-27 | NewtonFit Dev Sandbox deployed to dev-node-01 (VLAN 40) at dev-newtonfit.home.purvishome.com. Syncthing ingestion mounted and paired with S22 Ultra over VLAN boundaries via pfSense rules. stack-dev-finance.yml stack defined and build-and-push.ps1 script created. Finance container build blocked by cached package-lock.json Prisma 7 mismatch. |
| [[Session Notes — 2026-05-28 — Finance App Container Compilation Complete]] | 2026-05-28 | Docker Desktop backend started. Aligned lockfile via npm install --package-lock-only to Prisma 6.4.0. Resolved TypeScript type-checking failure in prisma.config.ts by adding fallbacks. Successfully compiled and published ghcr.io/scarredninja/webdev:latest to GHCR. Swarm stack dev-finance successfully deployed in Portainer and verified live at dev-finance.home.purvishome.com. |

---

## ➡️ Next Phase